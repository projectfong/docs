# Google Nest Thermostat GA02081-US - Home Assistant Matter Setup and Testing

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

This document records the installation, Matter commissioning, Home Assistant integration, network testing, Internet dependency testing, Google Home removal behavior, and recommissioning requirements observed with a Google Nest Thermostat model GA02081-US.

The thermostat can use Home Assistant through a direct local Matter CASE session over UDP 5540. However, testing on firmware 2.2-9 showed that completely blocking thermostat Internet access eventually caused its operational IPv6 and Matter state to fail. Restoring Internet access caused the Matter service to recover without another thermostat reboot.

Testing also established that removing Google's Matter fabric is different from removing the thermostat from Google Home. Removing only the Google Matter fabric preserves the Home Assistant Matter fabric, while removing the thermostat itself from Google Home caused the thermostat to leave the Home Assistant Matter fabric.

Google Home remains required for initial bootstrap and recommissioning. Commissioning also requires temporary Internet services that are broader than those observed during steady-state operation.

---

# Hardware

## Production Thermostat

Model:

```text
Google Nest Thermostat
GA02081-US
```

The production thermostat was selected for normal use.

---

## Cold Spare

A second GA02081-US is retained as a cold spare.

The cold spare was only powered from batteries to verify basic functionality.

It was intentionally not:

* Added to Google Home
* Connected to Wi-Fi
* Firmware updated
* Matter commissioned
* Added to Home Assistant

For long-term storage:

* Remove the AAA batteries.
* Keep the thermostat, base, documentation, and accessories together.
* Label it as the GA02081-US cold spare.

Suggested label:

```text
GA02081-US COLD SPARE
Hardware power-on test: PASS
Not commissioned
Batteries removed for storage
```

There is no reason to commission the spare until it is actually needed.

---

# HVAC Configuration

The existing thermostat was a Nest Learning Thermostat 3rd Generation.

The existing wiring was mapped to the corresponding GA02081-US terminals during installation.

| Function | Old Nest 3rd Gen | GA02081-US |
| -------- | ---------------- | ---------- |
| Cooling  | Y1               | Y          |
| Fan      | G                | G          |
| Heating  | W1               | W          |
| Common   | C                | C          |
| Power    | Rh               | R          |

The GA02081-US `*OB` terminal is unused.

The system is configured as a conventional heating and cooling system.

The installation has a C wire, so the thermostat has a stable HVAC power source in addition to its two AAA batteries.

---

# HVAC Safety

Before changing thermostat wiring:

1. Turn off the HVAC equipment using the local HVAC service switch.
2. Verify power is actually off with a multimeter.
3. Measure between R and C.
4. Confirm approximately 0 VAC before handling the conductors.

Do not depend on a non-contact voltage tester for 24 VAC HVAC wiring.

During installation, an accidental contact between a conductor and conductive mounting hardware reinforced the need to treat the wiring as energized until a meter proves otherwise.

Avoid shorting:

```text
R -> C
```

or:

```text
R -> chassis/ground
```

A direct R-C short can blow the low-voltage HVAC fuse.

---

# Initial Thermostat Configuration

After installation, complete the basic HVAC configuration on the thermostat and through Google Home.

For a furnace configuration where the equipment control board handles blower timing, when asked:

```text
Should the fan activate when calling for heat?
```

select:

```text
Don't activate
```

The furnace control board handles blower timing during a normal W heat call.

---

# Firmware

The thermostat initially contained:

```text
1.1-11
```

Google added Matter support to this thermostat family in:

```text
1.3-10
```

The production thermostat updated directly from:

```text
1.1-11
```

to:

```text
2.2-9
```

The installed version used throughout the Matter and Internet dependency testing was:

```text
2.2-9
```

Firmware history relevant to this setup:

| Version | Release date | Notes                      |
| ------- | ------------ | -------------------------- |
| 2.2-9   | 2026-03-24   | Bug fixes and improvements |
| 2.2-6   | 2025-08-04   | Bug fixes and improvements |
| 1.3-10  | 2023-06-12   | Matter support             |
| 1.1-11  | 2022-01-26   | Bug fixes and improvements |
| 1.1-9   | 2021-06-07   | Bug fixes and improvements |
| 1.0     | 2020-10-30   | Original release           |

---

# Updating the Thermostat

The thermostat requires Wi-Fi and Internet access to obtain firmware updates.

On the thermostat:

```text
Settings
  -> Version
  -> Check for update
```

During the update:

* Leave the thermostat connected to Wi-Fi.
* Allow Internet access.
* Preferably leave HVAC demand off.
* Allow the thermostat to reboot normally.

After updating, verify:

```text
Version: 2.2-9
```

After the firmware reboot, a compressor protection/start delay may appear before cooling starts.

This is normal behavior and is not a Matter failure.

---

# Google Home Bootstrap Requirement

The GA02081-US must first be added to Google Home before connecting it to a third-party Matter controller using the supported workflow tested here.

The required sequence is:

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
Share to Home Assistant
        |
        v
Home Assistant Matter fabric
```

Do not assume that a thermostat already running Matter-capable firmware can skip the Google Home bootstrap after a factory reset.

Matter-capable firmware does not eliminate the initial Google Home enrollment requirement observed and documented for this thermostat.

---

# Documentation and Testing Basis

It is important to distinguish platform requirements from observations made during this specific installation.

## Google Home Bootstrap

The initial supported configuration requires the Nest Thermostat to be enrolled through Google Home before sharing it with Home Assistant.

## Home Assistant Mobile Commissioning

The Home Assistant mobile Matter commissioning process uses the phone during commissioning.

Bluetooth can participate in commissioning even when the device later communicates over Wi-Fi.

The Home Assistant server's normal Bluetooth integration is not the same as the mobile Matter commissioning path.

## Observed GA02081-US Behavior

Several GA02081-US-specific operational behaviors were established during testing:

```text
Phone physically too far from thermostat:
Google Home could return "Can't set up device"

Phone physically close:
Matter handoff could proceed
```

Later recommissioning identified an additional requirement:

```text
Thermostat:
Settings -> Matter -> Manage

Leave thermostat on Matter Manage screen
while initiating:

Google Home
-> Linked Matter apps & services
-> Link apps & services
```

Both conditions were required during the successful recommissioning procedure:

```text
Phone physically close
+
Thermostat left on Matter -> Manage
```

These are installation observations and should not be represented as universally documented Google requirements unless independently confirmed for other firmware or devices.

---

# Network Configuration

## IoT VLAN

Network:

```text
<IO_T_VLAN>
```

IPv4 network:

```text
<IO_T_IPV4_NETWORK>
```

Thermostat IPv4 during testing:

```text
<THERMOSTAT_IPV4>
```

IPv6 prefix:

```text
<IO_T_IPV6_PREFIX>
```

The thermostat obtained IPv6 addresses within this prefix during operation.

---

# Matter Networking

Matter operational traffic observed for this thermostat used:

```text
UDP 5540
```

It is UDP, not TCP.

mDNS uses:

```text
UDP 5353
```

IPv4 multicast:

```text
224.0.0.251
```

IPv6 mDNS multicast:

```text
ff02::fb
```

The GA02081-US is a Wi-Fi Matter device.

It does not use the Home Assistant OpenThread Border Router for its normal Matter transport.

Bluetooth is part of the commissioning path, not the normal Home Assistant control path.

Normal operation is approximately:

```text
Home Assistant
        |
        | Local Matter CASE
        | UDP 5540
        v
GA02081-US
```

The later Internet dependency testing does not demonstrate that Home Assistant thermostat commands are cloud-routed.

The Home Assistant Matter connection remained a local CASE connection directly to the thermostat.

---

# Local Matter Firewall Policy

The desired internal security model is narrow access from the IoT network to Home Assistant rather than unrestricted IoT VLAN access to trusted networks.

Matter access to Home Assistant should permit:

```text
Source:
IoT VLAN / thermostat

Destination:
Home Assistant OS

Protocol:
UDP

Destination port:
5540
```

IPv6 Matter access is also permitted between the IoT IPv6 prefix and the HAOS IPv6 address.

A narrow IPv4 UDP/5540 rule was also added for completeness.

Existing Matter Server logs in this environment showed that Wi-Fi Matter devices could use IPv6 or IPv4 operational paths.

Do not create a TCP/5540 rule and assume it satisfies Matter.

---

# mDNS Reflection

Because the thermostat and HAOS are separated by VLAN boundaries, multicast discovery must be handled deliberately.

Avahi is used as the mDNS reflector.

During commissioning, the thermostat's commissionable Matter advertisement was visible using:

```bash
avahi-browse -rt _matterc._udp
```

An observed service was:

```text
<MATTER_COMMISSIONABLE_SERVICE_ID>
_matterc._udp
local
```

The thermostat advertised Matter on:

```text
5540
```

This proved that the thermostat was advertising its Matter commissioning service.

It did not prove that mobile commissioning, Bluetooth communication, Google Home multi-admin sharing, or Home Assistant fabric addition had succeeded.

Important:

```text
Avahi does not assign IPv6 addresses.
```

Avahi handles discovery and reflection of mDNS advertisements.

IPv6 addressing comes from the network's IPv6 configuration, router advertisements, SLAAC, DHCPv6, or equivalent network mechanisms.

---

# Firewall NAT Note

During troubleshooting, firewall logs showed traffic similar to:

```text
<THERMOSTAT_IPV4>:<source-port>
        ->
Google Internet address
```

with source NAT translating the thermostat to an internal translated address.

This did not mean the thermostat was trying to connect to the translated address.

It represented source NAT.

The logical flow was:

```text
Nest:
<THERMOSTAT_IPV4>:<port>
        |
        | SNAT
        v
<SNAT_ADDRESS>:<port>
        |
        v
Google Internet service
```

Return traffic was then destination-NATed back to:

```text
<THERMOSTAT_IPV4>
```

Do not mistake a translated NAT address for the Matter destination.

---

# Home Assistant Matter Prerequisites

Home Assistant OS already had a working Matter deployment before adding the thermostat.

Existing configuration included:

* Home Assistant OS
* Matter integration
* Home Assistant Matter Server
* Existing Matter devices
* Working local IPv6
* Working mDNS reflection
* Android Home Assistant Companion app

The existing Matter network was used to validate that the basic Home Assistant Matter infrastructure was functioning before troubleshooting the Nest.

---

# Android Commissioning Requirements

Matter commissioning from Home Assistant uses the phone during the commissioning process.

For Android:

* Use the full Home Assistant Companion app.
* Keep Google Play Services current.
* Enable Bluetooth.
* Keep Wi-Fi enabled.
* Allow the Home Assistant app appropriate Location permission.
* Keep the phone physically close to the thermostat.
* During the Google Home multi-admin handoff, leave the physical thermostat on its Matter Manage screen.

The distinction is:

```text
Normal operation:

Thermostat <-> Home Assistant
Local IP / Matter CASE / UDP 5540

Commissioning:

Phone <-> thermostat
Mobile Matter commissioning path
including Bluetooth
```

Successful IP connectivity to the thermostat does not eliminate the phone's role during commissioning.

---

# Initial Matter Commissioning to Home Assistant

Once the thermostat was:

* Installed
* Added to Google Home
* Connected to Wi-Fi
* Updated to firmware 2.2-9
* Showing Matter support

open the Home Assistant Companion app.

Navigate to:

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

Home Assistant displays instructions to share the existing Google Matter device with Home Assistant.

This uses Matter multi-admin rather than treating the thermostat as a new uncommissioned Matter device.

---

# Google Home Multi-Admin Sharing

In Google Home, open the Nest Thermostat.

Navigate to:

```text
Nest Thermostat
  -> Settings
  -> Linked Matter apps & services
  -> Link apps & services
```

Depending on the Google Home UI revision, `Linked Matter apps & services` may be under device information or settings.

Select:

```text
Home Assistant
```

Google Home then hands the Matter commissioning process to the Home Assistant Companion app.

Keep the phone physically close to the thermostat during this operation.

---

# Initial Critical Discovery: Phone Proximity

The first major commissioning failure occurred when attempting:

```text
Linked Matter apps & services
  -> Link apps & services
```

while physically too far from the thermostat.

Google Home returned:

```text
Can't set up device
```

No useful error code was displayed.

The Home Assistant Matter Server showed no corresponding commissioning transaction.

There was:

* No new node commissioning
* No PASE attempt
* No fabric addition
* No Matter Server commissioning failure

This initially made the problem appear potentially related to:

* Firewall rules
* IPv6 routing
* mDNS reflection
* VLAN routing
* Home Assistant Matter Server

Those were not the immediate cause of this failure.

After moving the Android phone physically close to the thermostat and repeating the Google Home Matter sharing operation, commissioning completed successfully.

The observed result was:

```text
Phone too far away:
Google Home -> "Can't set up device"

Phone physically close:
Google Home -> Home Assistant Matter handoff succeeds
```

Later recommissioning testing identified another required physical thermostat state, documented below.

---

# Successful Initial Commissioning Indicators

During successful commissioning, the thermostat displayed the Home Assistant Matter logo and:

```text
Linked
```

After commissioning, the thermostat's Matter menu showed:

```text
2 apps linked
```

The two Matter fabrics were:

```text
Google Home
Home Assistant
```

At this point Home Assistant could control the thermostat.

---

# Home Assistant Validation

Before changing Matter fabrics or firewall policies, verify normal thermostat operation from Home Assistant.

Test:

* Current temperature updates
* HVAC operating state
* Cooling setpoint
* Heating setpoint if applicable
* HVAC mode
* Changes made in HA appear on the thermostat
* Changes made locally on the thermostat appear in HA

Expected state immediately after initial multi-admin commissioning:

```text
Google Home Matter fabric:
ACTIVE

Home Assistant Matter fabric:
ACTIVE

Matter apps linked:
2
```

---

# Removing Only the Google Matter Fabric

After Home Assistant control was verified, Google's Matter fabric was removed directly from the thermostat.

On the thermostat, open the Matter fabric management screen.

The Google fabric appeared as:

```text
Google LLC
```

with an X/remove control.

Select the X.

The thermostat displays:

```text
Would you like
to disconnect
Google LLC?
```

Select:

```text
Remove
```

The thermostat confirms:

```text
Removed
```

This removes the Google Matter fabric without factory-resetting the thermostat.

Before removal:

```text
Matter
2 apps linked
```

After removal:

```text
Matter
1 app linked
```

The remaining Matter fabric was:

```text
Home Assistant
```

Home Assistant control remained functional immediately after this operation.

---

# Google Matter Fabric Is Not the Google Account Relationship

Later testing established an important distinction.

Removing:

```text
Google LLC
```

from the thermostat's Matter fabric list does not remove the thermostat from Google Home.

These are separate relationships.

After Google's Matter fabric had been removed, the thermostat still:

* Appeared in Google Home.
* Responded to Google Home commands.
* Maintained outbound Google/Nest network connections.
* Retained Home Assistant as its remaining Matter fabric.

The topology was effectively:

```text
Home Assistant
    |
    | Local Matter / CASE
    | UDP 5540
    v
GA02081-US
    |
    | Native Google/Nest
    | cloud relationship
    v
Google infrastructure
```

Therefore:

```text
Google Matter fabric removal
!=
Google account removal
!=
Google Home device removal
```

These operations must be treated separately.

---

# Important Google Home Matter Warning

Google Home also exposes an option approximately labeled:

```text
Unlink all Matter apps & services from this device
```

Do not use this when the goal is only to remove Google's Matter fabric.

Use the thermostat's specific per-fabric removal control for:

```text
Google LLC
```

when removing only that Matter fabric.

---

# Internet Isolation Test

The original goal was to determine whether the thermostat could operate as a completely local Home Assistant Matter device after commissioning.

The intended final policy was:

```text
Nest -> Internet
DENY

Nest -> HAOS Matter
ALLOW
```

Initial control after Internet blocking appeared functional.

Longer testing disproved the assumption that this represented a stable local-only configuration.

---

# Test 1: Block Thermostat Internet Access

Internet access from the thermostat was blocked while local network access remained available.

The purpose was to determine whether the GA02081-US could continue operating exclusively through local Matter after commissioning.

The thermostat at this stage had already been:

* Added to Google Home.
* Updated to firmware 2.2-9.
* Shared from Google Home to Home Assistant.
* Verified controllable from Home Assistant.
* Removed from the Google Matter fabric while retaining Home Assistant.

Home Assistant identified the thermostat using a Matter node and fabric specific to the local installation.

The operational Matter service used a locally generated Matter service identifier.

The thermostat hostname was locally generated.

The IPv4 address was:

```text
<THERMOSTAT_IPV4>
```

---

# Wi-Fi and IPv4 Remained Available

After Internet access was blocked, the thermostat remained reachable over IPv4.

Testing from another system on the IoT network using:

```text
ping <THERMOSTAT_IPV4>
```

continued to succeed.

The thermostat therefore had not simply disconnected from Wi-Fi.

---

# Matter Eventually Failed

Home Assistant eventually lost its active Matter subscription to the thermostat.

The Matter Server reported:

```text
All probes failed, closing session
```

followed by an offline state and eventually:

```text
offline grace period expired
```

Home Assistant repeatedly attempted to reconnect using changing IPv6 addresses associated with the thermostat.

The observed addresses were within:

```text
<IO_T_IPV6_PREFIX>
```

The Matter Server repeatedly reported:

```text
address is unreachable
```

---

# Direct IPv6 and Neighbor Discovery Testing

The system used for testing had an interface directly attached to the IoT IPv6 network.

Because both systems were on the same IPv6 prefix, direct Neighbor Discovery testing did not traverse the firewall/router.

An example route lookup was:

```bash
ip -6 route get <THERMOSTAT_IPV6>
```

The resulting path used the local IoT interface directly.

Neighbor Discovery entries for Nest operational addresses entered states including:

```text
FAILED
```

and:

```text
INCOMPLETE
```

Direct IPv6 pings returned:

```text
Destination unreachable: Address unreachable
```

This showed that the failure was not simply a routed firewall policy preventing HAOS from reaching the thermostat.

The operational ULA addresses being advertised or discovered for the Nest were not resolving successfully through IPv6 Neighbor Discovery.

---

# IPv6 Link-Local Remained Active

Packet capture showed that the thermostat had not lost IPv6 completely.

The Nest continued sending IPv6 multicast DNS traffic from its link-local address.

A captured announcement included the thermostat's operational `_matter._tcp.local` service.

At the same time, Neighbor Solicitations for advertised ULA addresses were not receiving usable Neighbor Advertisement responses.

The observed state was approximately:

```text
Wi-Fi:                   up
IPv4:                    up
IPv6 link-local:         up
mDNS:                    at least partially active
Operational IPv6 ULA:    unusable
Matter UDP 5540:         unusable
Home Assistant Matter:   offline
```

---

# Reboot While Internet Was Blocked

The thermostat was rebooted while Internet access remained blocked.

After reboot:

* The thermostat showed Wi-Fi connectivity when its Wi-Fi settings were opened.
* The operational Nest Matter service was no longer visible in `_matter._tcp`.
* Other Matter devices continued advertising normally.

The expected Nest Matter instance was absent.

Rebooting the thermostat while Internet remained blocked did not restore the Home Assistant Matter connection.

This disproved the earlier assumption that a normal reboot would necessarily recover Matter using only local network connectivity.

---

# Test 2: Restore Internet Without Rebooting

Internet access was restored without rebooting the thermostat again.

Almost immediately, Home Assistant discovered a new operational IPv6 address.

The Matter Server then recorded a new session with the thermostat over:

```text
UDP 5540
```

followed by the thermostat returning online.

A new Matter subscription completed successfully.

The Nest operational Matter service also reappeared in Avahi with a new operational IPv6 address.

Recovery occurred without another thermostat reboot.

---

# Internet Dependency Finding

Testing established the following behavior on this GA02081-US running firmware 2.2-9:

```text
Internet blocked
    |
    v
Thermostat remains on Wi-Fi
    |
    v
IPv4 remains reachable
    |
    v
Operational IPv6/Matter state eventually fails
    |
    v
Home Assistant Matter goes offline
    |
    v
Reboot while Internet remains blocked
does not restore operational Matter
    |
    v
Internet access restored
    |
    v
New operational IPv6 address appears
    |
    v
Matter service returns
    |
    v
Home Assistant establishes new CASE session
    |
    v
Thermostat becomes available
```

This does not demonstrate that normal Home Assistant commands are cloud-routed.

The active Home Assistant connection remained a local Matter CASE session directly to the thermostat on UDP 5540.

However, Internet connectivity was clearly involved in maintaining or recovering the thermostat's operational network/Matter state during this testing.

Therefore:

```text
Strict local-only operation:
NOT PROVEN

Internet-independent Matter operation:
DISPROVEN FOR THE TESTED CONFIGURATION
```

The earlier assumption that Internet access could simply be blocked permanently after Matter commissioning was incorrect.

---

# Restricted Internet Service Testing

Instead of leaving unrestricted outbound Internet access enabled, a restricted firewall service object was created for the thermostat.

The steady-state services initially tested were:

```text
TCP 443
TCP 11095
TCP 53
UDP 53
```

The thermostat remained stable for an extended testing period after Internet access was restored using this restricted service policy.

During that period there were no additional thermostat offline events, subscription failures, or operational-address failures in the Matter log.

Other Matter devices experienced unrelated connectivity events during the same period, while the Nest remained connected.

---

# Observed Nest Cloud Connection

Firewall logs showed the thermostat maintaining traffic to Google infrastructure using:

```text
TCP 11095
```

Session counters continued increasing rather than restarting from zero, indicating an ongoing or periodically active connection.

The thermostat was also still controllable from Google Home.

This reinforced the distinction between:

```text
Google Matter fabric
```

and:

```text
Native Google/Nest account and cloud relationship
```

Removing Google's Matter fabric did not remove the native cloud relationship.

---

# Test 3: Remove Thermostat From Google Home

To determine whether Google account membership could be removed while retaining Home Assistant Matter, the thermostat was removed from Google Home.

Google Home displayed:

```text
Remove this device?

This device will be removed from your home and unlinked from your Google Account.
```

The thermostat was removed.

No manual factory reset was performed.

---

# Google Home Removal Removed the Home Assistant Matter Fabric

The Home Assistant Matter Server received:

```text
basicInformation.leave
```

for the thermostat's Matter fabric.

Immediately afterward, the Matter peer left the fabric, the thermostat became unavailable, and the node was removed.

The thermostat itself also no longer showed the Home Assistant Matter link.

The observed sequence was:

```text
Remove thermostat from Google Home
    |
    v
Unlink thermostat from Google account
    |
    v
Thermostat generates Matter Leave event
    |
    v
Home Assistant fabric removed
    |
    v
Thermostat removed from Home Assistant Matter
```

This is an important operational distinction.

Do not use Google Home's:

```text
Remove device
```

action merely to eliminate Google cloud control after Home Assistant Matter has been established.

During this testing, that operation acted as a decommissioning action and removed the Home Assistant Matter relationship as well.

---

# Google Account Data Cleanup

Removing the thermostat from the Google Home structure was not the complete Google-side cleanup.

Old thermostat/account data also needed to be removed from the Google account.

This was necessary before recommissioning so the device did not remain associated with stale account information from the previous installation.

The cleanup sequence was:

1. Remove the thermostat from Google Home.
2. Go to the Google account.
3. Remove the old thermostat/account data.
4. Recommission the thermostat through Google Home.
5. Re-share the thermostat to Home Assistant using Matter.

This account-data cleanup step is important when rebuilding the thermostat after removing it from Google Home.

---

# Recommissioning Internet Requirements

The restricted steady-state firewall configuration was not sufficient for recommissioning.

During setup, firewall logs showed the thermostat attempting HTTP access to Google infrastructure.

Observed traffic included:

```text
<THERMOSTAT_IPV4> -> Google infrastructure
HTTP
TCP 80
```

Additional commissioning traffic included:

```text
<THERMOSTAT_IPV4> -> Google infrastructure
TCP 11095
```

DNS queries were also observed.

The commissioning Internet service policy therefore needs to include:

```text
TCP 80
TCP 443
TCP 11095
TCP 53
UDP 53
```

If TCP 80 is not allowed during commissioning, Google Home setup may fail to complete.

---

# TCP 80 Is Temporary

TCP 80 was observed as necessary during setup and recommissioning.

After commissioning completes, TCP 80 can be removed again.

The restricted steady-state service object can return to:

```text
TCP 443
TCP 11095
TCP 53
UDP 53
```

This should not be interpreted as proof that every one of these services is permanently required.

It represents the restricted steady-state policy that remained stable during the testing documented here.

---

# Critical Recommissioning Discovery: Matter Manage Screen

During recommissioning, another important Matter sharing requirement was identified.

Simply selecting:

```text
Linked Matter apps & services
```

followed by:

```text
Link apps & services
```

was not sufficient.

Google Home could return:

```text
Can't set up device
```

even though the thermostat supported Matter and was otherwise operating correctly.

The successful procedure required putting the physical thermostat into its Matter management screen first.

At the physical thermostat:

```text
Settings
  -> Matter
  -> Manage
```

Leave the thermostat sitting on:

```text
Matter -> Manage
```

Then, while remaining physically close to the thermostat, use Google Home:

```text
Nest Thermostat
  -> Linked Matter apps & services
  -> Link apps & services
  -> Home Assistant
```

Leaving the thermostat on the Matter Manage screen was required for reliable discovery and linking during this recommissioning test.

If the thermostat was not left on that screen, Google Home could return:

```text
Can't set up device
```

Phone proximity remained important as well.

The successful state was:

```text
Thermostat:
Matter -> Manage screen open

Phone:
Physically close to thermostat

Google Home:
Linked Matter apps & services
-> Link apps & services
-> Home Assistant
```

---

# Recommended Recommissioning Procedure

Use the following sequence based on the successful and failed testing performed.

## Step 1: Clean Up Old Google Account Data

If rebuilding after removing the thermostat from Google Home:

1. Remove stale thermostat/device information from Google Home as required.
2. Go to the Google account.
3. Remove the old thermostat/account data.

Do not assume that removing the device from the Google Home structure clears all old account information.

---

## Step 2: Temporarily Allow Commissioning Internet Services

Permit:

```text
Source:
Nest thermostat

Destination:
Internet

Services:
TCP 80
TCP 443
TCP 11095
TCP 53
UDP 53
```

Do not omit TCP 80 during commissioning.

---

## Step 3: Add Thermostat to Google Home

Add the thermostat through the normal Google Home workflow.

Complete:

* Wi-Fi configuration
* Google account enrollment
* HVAC setup as required
* Firmware update if required

Confirm that the thermostat is operational in Google Home.

---

## Step 4: Prepare Physical Thermostat for Matter Sharing

Go physically to the thermostat.

Open:

```text
Settings
  -> Matter
  -> Manage
```

Leave the thermostat on this screen.

Do not navigate away before completing the Home Assistant Matter sharing operation.

---

## Step 5: Keep Phone Close

Keep the Android phone physically close to the thermostat.

Verify:

* Bluetooth enabled
* Wi-Fi enabled
* Home Assistant Companion app installed
* Google Play Services current
* Required app permissions enabled

---

## Step 6: Start Home Assistant Matter Addition

In Home Assistant Companion:

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

---

## Step 7: Share From Google Home

While the thermostat remains on:

```text
Matter -> Manage
```

open Google Home and navigate to:

```text
Nest Thermostat
  -> Linked Matter apps & services
  -> Link apps & services
```

Select:

```text
Home Assistant
```

Complete the handoff.

---

## Step 8: Verify Home Assistant Matter

Confirm:

* Home Assistant Matter node appears.
* Thermostat is online.
* Current temperature updates.
* Setpoint changes work.
* HVAC mode changes work.
* Local thermostat changes propagate back to Home Assistant.

Do not change firewall policy until Home Assistant Matter control has been verified.

---

## Step 9: Remove TCP 80

After commissioning succeeds, remove:

```text
TCP 80
```

from the thermostat Internet service policy.

Return the tested restricted service object to:

```text
TCP 443
TCP 11095
TCP 53
UDP 53
```

---

## Step 10: Monitor Stability

Monitor:

* Matter Server logs
* Firewall sessions
* IPv6 operational addresses
* `_matter._tcp` advertisements
* Home Assistant entity availability
* Thermostat setpoint control
* State synchronization

Do not assume immediate success proves long-term stability.

---

# Steady-State Firewall Policy Under Test

The restricted Internet service object currently under test is:

```text
TCP 443
TCP 11095
TCP 53
UDP 53
```

Local Home Assistant Matter access remains:

```text
UDP 5540
```

The device should remain blocked from accessing trusted internal VLANs except for explicitly required local services.

Conceptually:

```text
Nest -> Trusted internal networks
DENY by default

Nest -> HAOS Matter
ALLOW UDP 5540

Nest -> Required discovery/local infrastructure
ALLOW as explicitly required

Nest -> Internet
RESTRICT to tested required services
```

This is not yet equivalent to a fully local-only configuration.

---

# Troubleshooting Decision Tree

## Symptom: Google Home Says "Can't set up device"

Before modifying the network, check the physical commissioning state.

At the thermostat:

```text
Settings
  -> Matter
  -> Manage
```

Leave it on the Manage screen.

Then:

```text
MOVE THE PHONE CLOSE TO THE THERMOSTAT
```

Retry:

```text
Google Home
  -> Linked Matter apps & services
  -> Link apps & services
  -> Home Assistant
```

Both thermostat state and phone proximity were important during the successful recommissioning procedure.

---

## Symptom: Google Home Initial Setup Will Not Complete

Verify the commissioning firewall policy includes:

```text
TCP 80
TCP 443
TCP 11095
TCP 53
UDP 53
```

TCP 80 was observed during commissioning and could not simply be omitted because it was unnecessary during steady-state testing.

---

## Symptom: Home Assistant Matter Server Shows No Commissioning Attempt

If the Matter Server logs show no:

```text
PASE
commissioning
new node
fabric addition
```

then the failure may be occurring before Home Assistant Matter Server participates.

Check:

1. Thermostat is on `Matter -> Manage`.
2. Phone is physically close to thermostat.
3. Bluetooth is enabled.
4. Home Assistant Companion app is being used.
5. Android Google Play Services are operational.
6. Google Home handoff is being initiated correctly.
7. Phone permissions are correct.
8. Commissioning Internet services are allowed.

Only after those should the Home Assistant Matter Server become the primary suspect.

---

## Symptom: Thermostat Advertises Matter But Cannot Be Added

Check mDNS:

```bash
avahi-browse -rt _matterc._udp
```

The Nest should advertise a `_matterc._udp` service while its commissioning window is open.

Remember:

```text
Visible _matterc._udp advertisement
        !=
Successful Bluetooth commissioning
        !=
Successful Google Home handoff
        !=
Successful Home Assistant fabric addition
```

Each stage validates a different part of the process.

---

## Symptom: Matter Stops Working After Internet Is Blocked

Check whether the thermostat still responds over IPv4:

```bash
ping <THERMOSTAT_IPV4>
```

Then inspect the Home Assistant Matter Server logs for:

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

Inspect IPv6 Neighbor Discovery and routing.

An IPv4-responsive thermostat does not prove that its operational Matter state remains usable.

During testing:

```text
IPv4:
UP

Wi-Fi:
UP

Matter:
FAILED

Operational IPv6 ULA:
FAILED
```

Restoring Internet caused the operational IPv6/Matter state to recover.

---

## Symptom: Matter Does Not Recover After Reboot With Internet Blocked

This behavior was observed during testing.

A reboot while Internet remained blocked did not restore the operational Matter service.

Restoring Internet without another reboot caused:

```text
New operational IPv6 address
        |
        v
Matter advertisement returns
        |
        v
New CASE session
        |
        v
Home Assistant node online
```

Do not assume a thermostat reboot is sufficient to recover this failure state.

---

## Symptom: Home Assistant Matter Disappears After Removing Device From Google Home

This was directly observed.

Google Home:

```text
Remove device
```

caused the thermostat to issue:

```text
basicInformation.leave
```

The Home Assistant Matter Server then removed the node.

Treat Google Home device removal as a decommissioning action unless future testing proves otherwise.

Do not use it merely to disable Google cloud connectivity.

---

## Symptom: Matter Works on Some Devices But Not Others

Matter can use IPv6 and IPv4 operational paths depending on device and network state.

Existing Matter Server logs in this environment showed:

* Successful IPv6 CASE sessions
* Failed or stalled IPv6 attempts followed by successful IPv4 CASE sessions
* Operational Wi-Fi Matter over IPv4 UDP 5540
* Thread Matter communication over IPv6

Therefore:

* Do not disable IPv6.
* Do not assume IPv4 is irrelevant.
* Do not assume every device selects the same operational path.
* Use Matter Server logs to determine the transport actually being used.

---

## Symptom: Firewall Logs Show an Unexpected Internal Address

Check whether the firewall is displaying a translated SNAT address.

For this setup:

```text
Actual Nest:
<THERMOSTAT_IPV4>
```

An address such as:

```text
<SNAT_ADDRESS>
```

after SNAT does not mean the thermostat is attempting to connect to that address.

Read the complete firewall session flow before modifying policies.

---

# Troubleshooting Layer Model

The process is easier to troubleshoot when separated into layers.

```text
Layer 1:
Thermostat hardware and HVAC configuration

Layer 2:
Thermostat Wi-Fi

Layer 3:
Commissioning Internet access

Layer 4:
Google Home bootstrap and account enrollment

Layer 5:
Matter-capable firmware

Layer 6:
Thermostat Matter Manage state

Layer 7:
Matter commissioning advertisement / mDNS

Layer 8:
Phone Bluetooth and mobile commissioning path

Layer 9:
Google Home multi-admin handoff

Layer 10:
Home Assistant Matter Server commissioning

Layer 11:
Operational Matter CASE session

Layer 12:
Long-term operational IPv6/Matter stability

Layer 13:
Restricted Internet operation
```

Do not treat success at one layer as proof that later layers work.

For example:

```text
Nest has Internet:
Does not prove Matter commissioning works.

_matterc._udp visible:
Does not prove Bluetooth commissioning works.

UDP 5540 allowed:
Does not prove Google Home completed the handoff.

HA Matter Server running:
Does not prove the commissioning request reached HAOS.

IPv4 ping works:
Does not prove operational Matter is healthy.

Initial local Matter control works:
Does not prove Internet-independent long-term operation.
```

---

# Things That Were Not the Initial Commissioning Problem

During the original Matter handoff troubleshooting, the following were investigated but were not the immediate cause of that failed sharing attempt:

* Nest Wi-Fi connectivity
* Nest Internet connectivity
* Firmware version after update
* Existing Google Matter fabric
* Maximum fabric count
* Home Assistant Matter Server availability
* Existing Home Assistant Matter devices
* Basic IPv6 connectivity
* IPv4 Matter transport
* Avahi seeing the Nest `_matterc._udp` advertisement
* Firewall SNAT address
* UDP 5540 firewall access

The immediate original failure was resolved by moving the phone physically close to the thermostat.

Later recommissioning established that leaving the thermostat on:

```text
Matter -> Manage
```

was also required for the reliable linking workflow observed during that test.

---

# Lessons Learned

## 1. Matter Commissioning Is Not Purely IP-Based

Even though the thermostat uses Wi-Fi for normal Matter operation, the mobile phone remains part of commissioning.

Seeing mDNS and UDP 5540 traffic does not prove that the mobile commissioning stage succeeded.

Commissioning transport and production transport are separate concepts.

---

## 2. Physical Proximity Matters

During the initial GA02081-US installation:

```text
Can't set up device
```

occurred while the phone was too far from the thermostat.

Moving the phone physically close allowed the handoff to complete.

Phone proximity should therefore be one of the first checks during Matter commissioning.

---

## 3. The Matter Manage Screen Matters During Recommissioning

Later testing established another requirement for the successful workflow.

Before selecting:

```text
Linked Matter apps & services
  -> Link apps & services
```

open:

```text
Settings
  -> Matter
  -> Manage
```

on the physical thermostat and leave it there.

The successful procedure required:

```text
Matter Manage screen
+
Phone physically close
+
Google Home Link apps & services
```

---

## 4. Home Assistant Matter Logs Identify the Failure Boundary

The absence of a commissioning attempt in the Home Assistant Matter Server logs was useful.

It showed that the transaction had not reached the expected Home Assistant commissioning stage.

That narrowed troubleshooting toward:

```text
Thermostat
    <- mobile commissioning ->
Phone
    <- Google Home handoff ->
Home Assistant Companion
```

rather than immediately changing the Matter Server or firewall.

---

## 5. Google Matter Fabric and Google Account Membership Are Different

Removing:

```text
Google LLC
```

from the thermostat's Matter fabric list preserved Home Assistant Matter.

It did not remove:

* Google Home device membership
* Google account association
* Google cloud control
* Native outbound Nest/Google connections

These are separate relationships.

---

## 6. Removing the Thermostat From Google Home Is Destructive to HA Matter

Google Home's:

```text
Remove device
```

operation caused the thermostat to leave the Home Assistant Matter fabric.

The Matter Server received:

```text
basicInformation.leave
```

and removed the node.

Do not use Google Home device removal as a cloud-disable switch after Home Assistant Matter commissioning.

---

## 7. Discovery Success Does Not Equal Commissioning Success

The command:

```bash
avahi-browse -rt _matterc._udp
```

can prove that a commissionable Matter advertisement exists.

It does not prove:

```text
Bluetooth commissioning succeeded
Google Home handoff succeeded
Home Assistant received the request
PASE completed
Home Assistant fabric was added
```

Validate each stage independently.

---

## 8. IPv4 Connectivity Does Not Prove Matter Health

During the Internet isolation failure:

```text
Wi-Fi:
UP

IPv4:
UP

IPv6 link-local:
UP

Operational IPv6 ULA:
UNUSABLE

Home Assistant Matter:
OFFLINE
```

The thermostat remained pingable over IPv4 while Home Assistant Matter was unavailable.

Do not use IPv4 ping alone as a Matter health check.

---

## 9. Restoring Internet Recovered Matter Without a Reboot

This was one of the strongest observations during testing.

The thermostat had already been rebooted while Internet was blocked and did not recover Matter.

After Internet was restored:

```text
New operational IPv6 address appeared
        |
        v
Matter service returned
        |
        v
Home Assistant established new CASE session
        |
        v
Thermostat became available
```

No additional thermostat reboot was required.

---

## 10. Local Matter Control Does Not Mean Internet Independence

Home Assistant communicated directly with the thermostat using a local Matter CASE session.

That establishes:

```text
HA command path:
LOCAL
```

It does not establish:

```text
Thermostat operational dependencies:
FULLY LOCAL
```

Testing showed that Internet connectivity was involved in maintaining or recovering the thermostat's operational Matter state.

These are separate questions.

---

## 11. TCP 80 Is Needed During Commissioning

A restricted firewall policy that was sufficient during steady-state testing was not sufficient for recommissioning.

Observed commissioning traffic required:

```text
TCP 80
TCP 443
TCP 11095
TCP 53
UDP 53
```

TCP 80 can be removed after commissioning.

---

## 12. TCP 11095 Is Actively Used

The thermostat maintained outbound traffic to Google infrastructure over:

```text
TCP 11095
```

This traffic was observed during commissioning and steady-state operation.

Its exact long-term necessity for Home Assistant Matter stability remains under investigation.

---

## 13. Troubleshoot the Earliest Failed Stage

The most effective diagnostic question remains:

```text
What is the earliest stage that did not occur?
```

Examples:

```text
Nest advertising:
YES

Google Home enrollment:
YES

Matter firmware:
YES

HA Matter Server commissioning request:
NO
```

Investigate the mobile handoff.

Another example:

```text
Wi-Fi:
YES

IPv4:
YES

Operational Matter:
NO

Operational IPv6:
NO

Internet:
BLOCKED
```

Investigate the operational network state and Internet dependency rather than assuming Wi-Fi failure.

---

# Current Proven State

The following behaviors were directly observed during testing:

* Home Assistant Matter commands to the thermostat use a local Matter connection.
* Matter operational traffic uses UDP 5540.
* The GA02081-US is a Wi-Fi Matter device.
* Google Home is required for the tested bootstrap/recommissioning workflow.
* Phone proximity is important during Matter sharing.
* Leaving the thermostat on `Matter -> Manage` was required during successful recommissioning.
* Removing Google's Matter fabric does not remove Google Home cloud control.
* Removing Google's Matter fabric preserves the Home Assistant Matter fabric.
* Completely blocking thermostat Internet access eventually caused operational Matter/IPv6 failure.
* IPv4 and Wi-Fi remained available during that failure.
* IPv6 link-local traffic remained present during that failure.
* Rebooting while Internet remained blocked did not restore operational Matter.
* Restoring Internet caused a new operational IPv6 address and Matter service to appear.
* Home Assistant automatically re-established a CASE session after Internet restoration.
* TCP 11095 traffic to Google infrastructure was observed.
* Removing the thermostat from Google Home also removed the Home Assistant Matter fabric.
* Old Google account/device data needed to be removed before recommissioning.
* TCP 80 was required during Google Home recommissioning.
* TCP 80 could be removed after commissioning.
* DNS was required during commissioning.
* TCP 11095 was active during commissioning.
* The thermostat must remain locally reachable by Home Assistant even when Internet policy is restricted.

---

# Items Still To Be Proven

The following questions remain open:

* Whether a different final configuration can maintain Home Assistant Matter indefinitely with Internet completely blocked.
* Whether TCP 11095 is required for long-term Matter stability or only native Google/Nest account connectivity.
* Whether TCP 443 is required for long-term Matter stability after recommissioning.
* Whether both TCP and UDP DNS are required during steady-state operation.
* Whether Google cloud association can be removed without triggering removal of the Home Assistant fabric.
* Whether a different Google account cleanup sequence can produce a fully local-only final state.
* Whether future firmware changes alter any of these behaviors.

Do not represent these unresolved questions as established facts.

---

# Current Production Architecture

The currently tested architecture is approximately:

```text
                    Local Matter
                    UDP 5540
                           |
                           |
                           v
Home Assistant OS <-> GA02081-US
Matter Server          Nest Thermostat
                           |
                           |
                           | Restricted Internet
                           v
                    Google/Nest services
```

Internal access remains restricted.

The desired security model is:

```text
Nest -> Trusted VLANs
DENY

Nest -> HAOS required Matter services
ALLOW

Nest -> Required local discovery/infrastructure
ALLOW as required

Nest -> Internet
RESTRICT
```

---

# Desired Final Objective

The long-term desired state remains:

```text
Google Matter fabric:
Removed

Home Assistant Matter:
Active

Home Assistant control:
Local

Trusted VLAN access:
Blocked

Internet access:
Blocked

Google cloud dependency:
None
```

That final state has not been proven stable.

The earlier assumption that Internet access could simply be blocked permanently after Matter commissioning was disproven during testing.

The currently proven distinction is:

```text
Home Assistant command path:
LOCAL

Thermostat completely Internet-independent:
NOT PROVEN
```

---

# Recovery Considerations

## Normal Reboot

A normal reboot does not inherently imply a factory reset.

However, testing showed that a normal reboot while Internet remained blocked did not restore the failed operational Matter state.

Do not use reboot success as an assumed recovery mechanism for Internet-isolated operation.

---

## Factory Reset

A factory reset is fundamentally different from a normal reboot.

Do not factory reset the production thermostat unless intentionally rebuilding it.

A rebuild should be expected to require the Google Home bootstrap again.

The recovery process is approximately:

```text
Factory reset / rebuild
        |
        v
Remove stale Google account data if present
        |
        v
Temporarily permit commissioning Internet services
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
        |
        v
Monitor Matter stability
```

---

# Production Thermostat State

```text
Model:
GA02081-US

Firmware:
2.2-9

Network:
<IO_T_VLAN>

IPv4 during testing:
<THERMOSTAT_IPV4>

Matter controller:
Home Assistant

Home Assistant transport:
Local Matter CASE / UDP 5540

Google Matter fabric:
Can be individually removed

Google Home account relationship:
Separate from Google Matter fabric

Internet:
Restricted during current testing

HVAC:
Conventional heating + cooling

C wire:
Installed

Home Assistant control:
Working when operational Matter state is healthy
```

---

# Cold Spare State

```text
Model:
GA02081-US

Condition:
Cold spare

Hardware test:
Passed

Google Home:
Not configured

Wi-Fi:
Not configured

Matter:
Not commissioned

Firmware:
Not intentionally updated

Role:
Cold spare
```

The spare remains untouched until required.

---

# Complete Tested Workflow

The complete installation and testing path was:

```text
Existing thermostat removed
        |
        v
GA02081-US installed
        |
        v
Google Home bootstrap
        |
        v
Firmware 1.1-11 -> 2.2-9
        |
        v
Matter enabled
        |
        v
Google Home Matter fabric active
        |
        v
Phone moved physically close
        |
        v
Google Home -> Link apps & services
        |
        v
Home Assistant fabric added
        |
        v
2 apps linked
        |
        v
HA control verified
        |
        v
Google LLC Matter fabric removed
        |
        v
Home Assistant Matter remained active
        |
        v
Internet blocked
        |
        v
Initial local control appeared functional
        |
        v
Operational IPv6/Matter eventually failed
        |
        v
IPv4 remained reachable
        |
        v
Reboot with Internet blocked
        |
        v
Matter did not recover
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
HA CASE session restored
        |
        v
Restricted Internet services tested
        |
        v
Google Home cloud relationship confirmed separate
from Google Matter fabric
        |
        v
Thermostat removed from Google Home
        |
        v
Matter Leave event generated
        |
        v
Home Assistant Matter fabric removed
        |
        v
Old Google account data removed
        |
        v
Recommissioning started
        |
        v
TCP 80 found necessary during setup
        |
        v
Thermostat placed on Matter -> Manage
        |
        v
Phone kept physically close
        |
        v
Google Home -> Link apps & services
        |
        v
Home Assistant Matter sharing succeeds
        |
        v
TCP 80 removable after commissioning
        |
        v
Restricted steady-state Internet testing continues
```

---

# Most Important Operational Takeaways

For Matter commissioning:

```text
Keep the phone physically close to the thermostat.
```

For Google Home to Home Assistant Matter sharing:

```text
Open the physical thermostat:

Settings -> Matter -> Manage

Leave it on that screen while using:

Google Home
-> Linked Matter apps & services
-> Link apps & services
-> Home Assistant
```

For commissioning firewall access:

```text
Allow temporarily:

TCP 80
TCP 443
TCP 11095
TCP 53
UDP 53
```

After commissioning:

```text
TCP 80 can be removed.
```

For Google relationship management:

```text
Removing Google Matter fabric
does not remove Google Home account/cloud relationship.
```

Do not confuse that with:

```text
Google Home -> Remove device
```

because during testing that action caused:

```text
Matter Leave
        |
        v
Home Assistant fabric removed
```

For Internet isolation:

```text
Local HA Matter commands:
VERIFIED

Completely Internet-independent GA02081-US operation:
NOT VERIFIED
```

Blocking Internet completely eventually caused operational Matter failure during the documented test.

Restoring Internet recovered the operational IPv6 address and Home Assistant CASE session without another thermostat reboot.

---

# Related Search Keywords

GA02081-US Google Nest Thermostat, Home Assistant Matter, Google Home multi-admin, Matter UDP 5540, Avahi mDNS _matterc._udp, Nest Internet dependency, Nest local control, Google LLC Matter fabric remove, Nest Thermostat 2.2-9, Home Assistant CASE session

---

## Revision Control

| Version   | Date       | Summary                                                                                                                                                                                                                        | Author      |
| --------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------- |
| **1.0.0** | 2026-09-13 | Initial merged publication documenting GA02081-US installation, Home Assistant Matter commissioning, Internet dependency testing, Google Home removal behavior, firewall requirements, and verified recommissioning procedure. | projectfong |
