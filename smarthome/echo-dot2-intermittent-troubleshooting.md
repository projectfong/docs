# Echo Dot 2 EchoMuse Periodic Wi-Fi Disconnect Investigation

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

This document records the investigation into recurring EchoMuse controller disconnects on a repurposed Amazon Echo Dot 2. Testing confirms that the primary recurring disconnect is initiated by the Echo Dot's Fire OS `WifiDiagsUtil` network-bounce mechanism after connectivity diagnostics report `BAD_CONNECTION`, rather than by weak Wi-Fi signal or a confirmed wireless infrastructure failure. A built-in runtime control for network bounce has been identified, but the exact Intent extra required to disable the behavior has not yet been determined or validated.

---

## Purpose

This investigation determines why the repurposed Echo Dot 2 periodically loses its EchoMuse controller connection and identifies a remediation that does not unnecessarily modify the wireless network, `wpa_supplicant`, MediaTek Wi-Fi driver, or EchoMuse.

The investigation has the following goals:

* Identify whether the disconnect originates from the Echo Dot, EchoMuse, wireless infrastructure, or RF conditions.
* Correlate EchoMuse controller disconnects with Fire OS network events.
* Determine whether the recurring disconnect is random or timer-driven.
* Identify the Fire OS component initiating the disconnect.
* Determine whether the responsible behavior can be disabled through an existing Fire OS control.
* Preserve verified commands, logs, and findings for continued troubleshooting.
* Separate the primary recurring Wi-Fi disconnect from shorter controller interruptions that have not yet been correlated with the same cause.

Estimated time to continue the current investigation:

```text
Framework inspection:       15-30 minutes
Runtime disable test:        5-10 minutes
Initial validation:          5-10 minutes
Full timer validation:       45-60 minutes
```

The full validation period is required because an observed network-bounce sequence repeated after approximately 42 minutes.

**Next Step:** Determine the literal Intent extra used by `WifiDiagsConfigReceiver` for the `SET_ENABLENETBOUNCE` action.

---

## Scope

This document covers the recurring Wi-Fi and EchoMuse controller disconnect behavior observed on the repurposed Echo Dot 2.

Included:

* EchoMuse controller connection state.
* Echo Dot Wi-Fi state.
* MediaTek MT8163 wireless behavior.
* Fire OS connectivity diagnostics.
* `WifiDiagsUtil`.
* `system_server`.
* `fosservices.odex`.
* Amazon network-bounce behavior.
* Runtime network-bounce configuration discovery.
* Commands used during troubleshooting.
* Current remediation options.
* Validation requirements.

Not currently included:

* General Echo Dot 2 rooting procedures.
* Initial EchoMuse installation.
* Home Assistant configuration unrelated to this failure.
* General wireless access point configuration.
* Unrelated Echo Dot services.
* Short controller disconnects unless they can be correlated with the same network-bounce mechanism.

**Validation Result:** The investigation remains scoped to the observed disconnect behavior and the components directly involved in reproducing or explaining it.

---

## Sanitization Reference

The following documentation-only values are used throughout this public-safe version.

| Purpose | Documentation Value |
| --- | --- |
| Home Assistant IPv4 network | `192.0.2.0/24` |
| IoT IPv4 network | `198.51.100.0/24` |
| Home Assistant controller | `192.0.2.54` |
| Echo Dot 2 | `198.51.100.60` |
| IoT network gateway | `198.51.100.1` |
| IoT SSID | `iot-matter` |
| Wireless access point BSSID | `02:00:00:00:00:01` |

The IPv4 addresses use documentation ranges and do not represent production addressing.

The BSSID is a locally administered example identifier and does not represent a production access point.

**Validation Result:** Environment-specific network identifiers have been replaced with public-safe documentation values.

---

## Core Components

| Component | Description | Notes |
| --- | --- | --- |
| Echo Dot 2 | Repurposed Amazon voice hardware running legacy Fire OS. | Affected device. |
| EchoMuse | Provides the repurposed voice functionality and controller connection. | Server remains present during investigated events. |
| `wlan0` | Primary Echo Dot wireless interface. | Managed by the Fire OS Wi-Fi stack. |
| MediaTek MT8163 | Wi-Fi platform identified through Fire OS properties. | Driver records software-requested disassociation. |
| `wpa_supplicant` | Handles WPA2 wireless association. | Association is healthy during normal operation. |
| `dhcpcd` | Provides IPv4 network configuration. | Valid configuration observed during testing. |
| `system_server` | Android system process containing the active Amazon Wi-Fi diagnostics. | `WifiDiagsUtil` executes here. |
| `WifiDiagsUtil` | Amazon Fire OS connectivity diagnostic component. | Initiates `networkBounce` after failed diagnostics. |
| `fosservices.odex` | Amazon Fire OS framework containing `WifiDiagsUtil`. | Contains network-bounce controls and configuration receiver. |
| Wireless access point | Provides wireless connectivity for the Echo Dot. | Strong signal observed during investigation. |
| Home Assistant controller | EchoMuse controller endpoint. | Controller connection is interrupted when Echo Wi-Fi is bounced. |

**Validation Result:** The responsible diagnostic logic has been traced to Amazon Fire OS code executing inside `system_server`.

---

## Environment

### Echo Dot Platform

Fire OS reports the following Wi-Fi platform information:

```text
[mediatek.wlan.chip]: [CONSYS_MT8163]
[mediatek.wlan.module.postfix]: [_consys_mt8163]
[ro.mediatek.wlan.p2p]: [1]
[ro.mtk_wlan_support]: [1]
[wifi.interface]: [wlan0]
[wlan.driver.status]: [ok]
```

The primary networking processes observed during troubleshooting include:

```text
/system/bin/wpa_supplicant
/system/bin/dhcpcd
```

Example process output:

```text
wifi      25717 1     16092 3664  ... /system/bin/wpa_supplicant
dhcp      26901 1     10072 744   ... /system/bin/dhcpcd
```

### EchoMuse Server

The EchoMuse server was observed running as:

```text
/data/local/bin/server
```

Example:

```text
root  1147  ... /data/local/bin/server
```

Open resources associated with the server included:

```text
/dev/null
/tmp/server.log
/dev/snd/pcmC0D23p
/dev/snd/pcmC0D24c
/dev/input/event1
/dev/input/event2
```

No observed evidence during this investigation shows the EchoMuse server process terminating as the initiating event.

### Shell Access

Direct ADB access is no longer being used.

Root shell access remains available through the EchoMuse console.

This shell is sufficient to access:

```text
logcat
getprop
wpa_cli
ip
ps
settings
am
/proc
/system/framework
```

BusyBox is also installed:

```text
BusyBox v1.22.1
```

**Validation Result:** The EchoMuse console provides sufficient root access to continue troubleshooting and test a runtime remediation.

---

## Wireless Baseline

### Signal Strength

The Echo Dot was moved close to the wireless access point during troubleshooting.

Wireless infrastructure observations included approximately:

```text
RSSI: -57 dBm
SNR:   38 dB
```

and later:

```text
RSSI: -51 dBm
SNR:   44 dB
```

The Echo Dot itself reported:

```text
[wifi.ro.wlan0.rssi]: [-41]
```

These measurements show strong RF conditions during the investigation.

A Home Assistant Voice Preview device located beside the Echo Dot and connected to the same SSID did not exhibit the same Wi-Fi disconnect behavior.

### Wi-Fi Association

The correct `wpa_cli` control socket is:

```text
/data/misc/wifi/sockets
```

Use:

```bash
wpa_cli -i wlan0 -p /data/misc/wifi/sockets status
```

Sanitized example output:

```text
bssid=02:00:00:00:00:01
freq=2412
ssid=iot-matter
id=0
mode=station
pairwise_cipher=CCMP
group_cipher=CCMP
key_mgmt=WPA2-PSK
wpa_state=COMPLETED
ip_address=198.51.100.60
```

The `2412` MHz frequency corresponds to 2.4 GHz channel 1.

The important state is:

```text
wpa_state=COMPLETED
```

### Interface State

Use:

```bash
ip addr show wlan0
```

The interface was observed as:

```text
wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP>
state UP
```

The interface contained:

* A valid IPv4 address.
* Global IPv6 addresses.
* A link-local IPv6 address.

### Routing

Use:

```bash
ip route
```

Sanitized example:

```text
default via 198.51.100.1 dev wlan0
```

A directly connected local route was also present.

**Validation Result:** Normal Wi-Fi association, interface configuration, routing, and RF conditions are healthy between disconnect events.

---

## Problem Symptoms

EchoMuse periodically reports:

```text
controllerDisconnected
```

followed by:

```text
controllerConnected
```

One longer observed event was:

```text
9:53:50 AM  controllerDisconnected
10:02:37 AM controllerConnected
```

The controller recovery took approximately nine minutes.

During that period, packet capture showed repeated mDNS queries from the controller side for:

```text
_emcontroller._tcp.local
```

This demonstrated that discovery attempts continued while EchoMuse remained disconnected.

Other events recovered considerably faster.

Example:

```text
11:48:32 PM controllerDisconnected
11:48:37 PM controllerConnected
```

This event lasted approximately five seconds.

**Next Step:** Correlate controller events with Fire OS diagnostic and Wi-Fi state transitions rather than treating controller recovery duration as the initiating failure.

---

## Cause Investigation

### Initial Cause Considered: Weak Wi-Fi

The first suspected cause was unstable RF connectivity from the Echo Dot's older Wi-Fi hardware.

This became less likely after:

* Moving the Echo Dot close to the wireless access point.
* Observing approximately `-41 dBm` from the Echo.
* Observing strong access point RSSI and SNR.
* Confirming another nearby device on the same SSID remained connected.
* Observing the Echo reconnect to the same access point after a disconnect.

### Cause

Available evidence does not support weak Wi-Fi signal as the primary recurring failure.

### Fix

No RF-related configuration change was made because the available evidence did not support weak signal as the root cause.

**Validation Result:** Weak RF conditions are not supported as the cause of the primary recurring disconnect.

---

## Fire OS Connectivity Diagnostics

Android `logcat` identified an Amazon-specific component:

```text
WifiDiagsUtil
```

This component performs connectivity tests that include:

* Localhost connectivity.
* Gateway connectivity.
* DNS or nameserver connectivity.
* HTTP connectivity.
* Captive portal detection.

Observed Internet resources included:

```text
fos5echocaptiveportal.com
d3p8zr0ffa9t17.cloudfront.net
2.android.pool.ntp.org
```

These are external service names observed in Fire OS diagnostic activity and are retained because they are not private infrastructure identifiers.

Diagnostic failures included unsuccessful gateway and nameserver tests and DNS resolution failures.

Fire OS subsequently reported:

```text
BAD_CONNECTION
```

### Cause

The Echo Dot can have a valid Wi-Fi association and strong RF signal while Amazon's higher-level Fire OS connectivity diagnostics consider the network unusable.

### Fix

No connectivity-test bypass has been applied.

The preferred remediation is to disable the network-reset response rather than change the local network solely to satisfy legacy Fire OS connectivity tests.

**Validation Result:** `BAD_CONNECTION` is generated by Fire OS connectivity diagnostics even while the underlying Wi-Fi association is otherwise operational.

---

## Network Bounce Discovery

The critical Fire OS events include:

```text
event=BAD_CONNECTION
```

followed by:

```text
event=networkBounce,action=newNetBounceSequence
```

Additional observed events include:

```text
event=networkBounceFailure
```

```text
event=networkBounceSuccess
```

```text
event=maxNetworkBounceAttemptsExceeded, action=disableNetworkBounce
```

```text
event=disableNetworkBounceTimeoutActive, action=null
```

```text
event=disableNetworkBounceTimeoutExpired, action=performNetworkBounce
```

These messages establish that `networkBounce` is an explicit Fire OS recovery mechanism.

### Cause

Fire OS responds to failed connectivity diagnostics by deliberately attempting to reset the network connection.

### Fix

The desired remediation is to disable `networkBounce` while leaving the underlying Wi-Fi connection and EchoMuse service operational.

**Validation Result:** The recurring Wi-Fi reset is associated with an explicit Amazon Fire OS network-recovery mechanism.

---

## Driver-Level Confirmation

Kernel logging captured:

```text
wlanoidSetDisassociate
DisconnectByOid
```

This was followed by:

```text
wlan0 netif_carrier_off
```

The MediaTek Wi-Fi stack then scanned wireless channels and reassociated.

The Echo subsequently returned to the same wireless access point.

Observed reconnect signal strength was approximately:

```text
Connect Rssi = -40
```

The resulting sequence is:

```text
Strong Wi-Fi association
        |
        v
Fire OS connectivity diagnostics
        |
        v
BAD_CONNECTION
        |
        v
networkBounce
        |
        v
software-requested disassociation
        |
        v
DisconnectByOid
        |
        v
wlan0 carrier off
        |
        v
wireless scan
        |
        v
same wireless access point found
        |
        v
reassociation
```

### Cause

The MediaTek driver receives a software-requested disassociation instead of simply reporting loss of the access point.

### Fix

Do not modify the MediaTek driver. The appropriate remediation is higher in the Fire OS stack where `networkBounce` originates.

**Validation Result:** Driver logging supports a software-initiated disconnect rather than an unexplained RF loss.

---

## Responsible Process Identification

The following command was used:

```bash
logcat -d -v threadtime | busybox grep 'WifiDiagsUtil'
```

`WifiDiagsUtil` was observed under PID:

```text
1644
```

The process was identified using:

```bash
busybox cat /proc/1644/cmdline
```

Result:

```text
system_server
```

The executable was:

```text
/system/bin/app_process32
```

Process mappings included:

```text
/system/framework/arm/fosservices.odex
/system/framework/arm/wifi-service.odex
/system/framework/arm/services.odex
```

### Cause

`WifiDiagsUtil` is not an independent application process. It executes inside Android `system_server`.

### Fix

Do not terminate `system_server`.

Troubleshooting must target the Amazon Wi-Fi diagnostic configuration or implementation instead.

**Validation Result:** PID and `/proc` inspection place the responsible diagnostic code inside `system_server`.

---

## Fire OS Framework Identification

String inspection of:

```text
/system/framework/arm/fosservices.odex
```

identified:

```text
com/amazon/android/service/wifidiags/WifiDiagsUtil
```

The framework also contains:

```text
WifiDiagsConfigReceiver
```

and network-bounce-related methods and fields.

Observed symbols include:

```text
checkNetworkBounceSuccess
onNetworkBounceTimeout
mDisableNetBounceFlag
mDisableNetBounceTime
mEnableNetBounce
mLastNetBounceBssid
mLastNetBounceTime
mNetBounceCounter
mNetBounceStartTime
mNetworkBounceInProgress
```

Timing and state constants include:

```text
NETBOUNCE_DEFAULT_START_TIME
NETBOUNCE_DISABLE_SECS
NETBOUNCE_MAX_ATTEMPTS
NETBOUNCE_MAX_COUNTER_RESET_SECS
NETBOUNCE_MIN_INTERVAL_SECS
```

### Cause

The Amazon-specific network-bounce implementation resides in the Fire OS framework rather than in EchoMuse.

### Fix

Use the framework's existing runtime control before considering any modification to `fosservices.odex`.

**Validation Result:** `fosservices.odex` contains the identified Amazon Wi-Fi diagnostic and network-bounce implementation.

---

## Network-Bounce State Machine

Captured events show a retry and cooldown mechanism.

The observed behavior can be represented as:

```text
BAD_CONNECTION
        |
        v
networkbounceCounter=1
        |
        v
networkBounceFailure
        |
        v
networkbounceCounter=2
        |
        v
networkBounceFailure
        |
        v
networkbounceCounter=3
        |
        v
networkBounceFailure
        |
        v
maxNetworkBounceAttemptsExceeded
        |
        v
disableNetworkBounce
        |
        v
disableNetworkBounceTimeoutActive
        |
        v
cooldown period
        |
        v
disableNetworkBounceTimeoutExpired
        |
        v
performNetworkBounce
```

During the disabled period, Fire OS continues to report:

```text
BAD_CONNECTION
```

without immediately resetting the network.

This distinction is important.

The Echo can continue operating while Fire OS considers the network bad. The disruptive behavior is the subsequent network-bounce recovery action.

**Validation Result:** Fire OS already contains a state in which connectivity diagnostics continue while network resets are suppressed.

---

## Timing Correlation

A captured sequence began at:

```text
13:16:37.998
event=BAD_CONNECTION
```

Immediately afterward:

```text
13:16:37.999
event=networkBounce,action=newNetBounceSequence,networkbounceCounter=1
```

The sequence continued:

```text
13:19:38.032
event=networkBounceFailure
```

```text
13:21:03.126
event=networkBounce,action=continueNetBounceSequence,networkbounceCounter=2
```

```text
13:24:03.133
event=networkBounceFailure
```

```text
13:25:28.221
event=networkBounce,action=continueNetBounceSequence,networkbounceCounter=3
```

```text
13:28:28.224
event=networkBounceFailure
```

Fire OS then continued reporting `BAD_CONNECTION` without performing another immediate bounce.

A new sequence started at:

```text
13:58:55.366
event=BAD_CONNECTION
```

followed by:

```text
13:58:55.367
event=networkBounce,action=newNetBounceSequence,networkbounceCounter=1
```

The interval between:

```text
13:16:37
```

and:

```text
13:58:55
```

is approximately:

```text
42 minutes 18 seconds
```

An independently observed EchoMuse event occurred at:

```text
11:48:32 PM controllerDisconnected
11:48:37 PM controllerConnected
```

This occurred within seconds of the previously expected recurring boundary during that observation period.

### Cause

The primary recurring disconnect follows the Fire OS network-bounce retry and cooldown behavior rather than occurring randomly.

### Fix

The validation target for a remediation must include remaining connected through at least one complete expected network-bounce interval.

**Validation Result:** The approximately 42-minute recurrence closely correlates with the Fire OS network-bounce state machine.

---

## Runtime Configuration Interface

Inspection of `fosservices.odex` identified runtime Intent controls for the Wi-Fi diagnostic subsystem.

Observed action names include:

```text
SET_ENABLECAPTIVEPORTALCHECK
SET_ENABLENETBOUNCE
SET_ENABLEWIFIDIAGS
SET_PERFORMHTTPTEST
SET_PERFORMPINGTEST
```

The full network-bounce action is:

```text
com.amazon.android.service.wifidiags.SET_ENABLENETBOUNCE
```

Related internal fields include:

```text
INTENT_EXTRA_ENABLECAPTIVEPORTALCHECK_FIELD
INTENT_EXTRA_ENABLENETBOUNCE_FIELD
INTENT_EXTRA_ENABLEWIFIDIAGS_FIELD
INTENT_EXTRA_METRICSINTERVAL_FIELD
INTENT_EXTRA_PERFORMHTTPTEST_FIELD
INTENT_EXTRA_PERFORMPINGTEST_FIELD
INTENT_EXTRA_PRECONFIGUREDEXTNS_FIELD
INTENT_EXTRA_TESTINTERVALMINUTES_FIELD
INTENT_EXTRA_TRIGGERNETRECOVERY_FIELD
```

The framework also contains:

```text
event=receivedIntentSetEnableNetBounce, val=
```

and:

```text
event=networkBounceDisabled, action=skipResetConnection
```

### Cause

Amazon implemented an explicit runtime configuration path for enabling or disabling network bounce.

### Fix

Determine the literal value represented by:

```text
INTENT_EXTRA_ENABLENETBOUNCE_FIELD
```

and send the appropriate boolean value through the existing broadcast receiver.

**Validation Result:** A built-in runtime network-bounce control exists and is preferable to patching the Fire OS framework.

---

## Broadcast Receiver Validation

The receiver was tested using:

```bash
am broadcast -a com.amazon.android.service.wifidiags.SET_ENABLENETBOUNCE
```

Android returned:

```text
Broadcasting: Intent { act=com.amazon.android.service.wifidiags.SET_ENABLENETBOUNCE }
Broadcast completed: result=0
```

`logcat` subsequently reported:

```text
event=receivedIntentSetEnableNetBounce, val=true
```

This confirms:

1. The action name is valid.
2. The broadcast reaches `WifiDiagsConfigReceiver`.
3. The receiver evaluates the network-bounce enable value.
4. The value defaults to `true` when the expected extra is not supplied.

### Failed Extra Test

The following was tested:

```bash
am broadcast \
    -a com.amazon.android.service.wifidiags.SET_ENABLENETBOUNCE \
    --ez enableNetBounce false
```

Android accepted the broadcast:

```text
Broadcast completed: result=0
```

However, `WifiDiagsUtil` continued reporting:

```text
event=receivedIntentSetEnableNetBounce, val=true
```

Therefore:

```text
enableNetBounce
```

is not the correct Intent extra name.

### Cause

The broadcast action is valid, but the tested `enableNetBounce` boolean extra is not the key expected by the receiver.

### Fix

Do not continue guessing extra names.

Inspect `WifiDiagsConfigReceiver.onReceive()` to determine the exact string passed to `Intent.getBooleanExtra()`.

**Validation Result:** The broadcast action works, but the correct boolean extra remains unresolved.

---

## Android Settings Investigation

The legacy Fire OS `settings` command supports:

```text
settings get
settings put
settings delete
```

It does not support:

```text
settings list
```

Candidate settings were queried across `global`, `secure`, and `system`.

Examples:

```bash
settings get global network_bounce_enabled
settings get secure network_bounce_enabled
settings get system network_bounce_enabled
```

Additional names included:

```text
wifi_network_bounce_enabled
wifi_diags_enabled
wifi_diagnostics_enabled
captive_portal_detection_enabled
```

The tested values returned:

```text
null
```

System properties were also searched:

```bash
getprop | busybox grep -i -E 'bounce|diag'
```

No confirmed configuration property controlling network bounce was identified.

### Cause

No tested Android SettingsProvider key or system property exposes the required control.

### Fix

Continue using the confirmed Amazon-specific Intent interface rather than adding unverified Android settings.

**Validation Result:** No currently tested SettingsProvider or system-property control provides the required network-bounce setting.

---

## Primary Root Cause

The verified primary failure path is:

```text
Echo Dot connected to wireless access point
        |
        | strong RF
        | WPA2 COMPLETED
        | valid network configuration
        |
        v
Amazon WifiDiagsUtil
        |
        v
connectivity tests fail
        |
        v
BAD_CONNECTION
        |
        v
network-bounce state machine
        |
        v
networkBounce
        |
        v
software-requested Wi-Fi disassociation
        |
        v
MediaTek DisconnectByOid
        |
        v
wlan0 carrier off
        |
        v
wireless scan
        |
        v
same access point rediscovered
        |
        v
Wi-Fi reassociation
        |
        v
EchoMuse controller connection recovers
```

### Cause

The primary recurring EchoMuse disconnect is caused by Amazon Fire OS deliberately resetting the Echo Dot's Wi-Fi connection after its connectivity diagnostics classify the network as `BAD_CONNECTION`.

### Fix

Disable the Fire OS network-bounce recovery action using the existing `SET_ENABLENETBOUNCE` runtime interface once its required boolean Intent extra is identified.

**Validation Result:** The initiating component and failure sequence have been identified. The remediation mechanism exists but has not yet been successfully activated.

---

## Separate Short Controller Interruptions

Not every EchoMuse controller disconnect observed during testing follows the identified network-bounce timing pattern.

Examples include:

```text
11:28:56 PM controllerDisconnected
11:28:57 PM controllerConnected
```

and:

```text
12:02:23 AM controllerDisconnected
12:02:24 AM controllerConnected
```

These interruptions lasted approximately one second.

They have not yet been correlated with:

```text
BAD_CONNECTION
networkBounce
DisconnectByOid
```

### Cause

Unknown.

Available evidence is insufficient to assign these short controller interruptions to the Fire OS network-bounce mechanism.

### Fix

Do not modify anything specifically for these events until they are captured alongside Fire OS and EchoMuse logs.

If they continue after network bounce is disabled, investigate them separately.

**Validation Result:** Short controller interruptions remain a separate unresolved observation.

---

## Preferred Fix

The preferred remediation is to use Amazon's existing runtime configuration interface.

Known action:

```text
com.amazon.android.service.wifidiags.SET_ENABLENETBOUNCE
```

The final command is expected to follow this structure:

```bash
am broadcast \
    -a com.amazon.android.service.wifidiags.SET_ENABLENETBOUNCE \
    --ez <EXACT_INTENT_EXTRA> false
```

The literal value for:

```text
<EXACT_INTENT_EXTRA>
```

has not yet been identified.

Do not use:

```text
enableNetBounce
```

because testing confirmed that this key does not change the value.

### Required Success Log

The first required validation is:

```text
event=receivedIntentSetEnableNetBounce, val=false
```

The next required validation is:

```text
event=networkBounceDisabled, action=skipResetConnection
```

The following should no longer be initiated by the diagnostic failure:

```text
event=networkBounce,action=newNetBounceSequence
```

### Expected Behavior

```text
WifiDiagsUtil
        |
        v
BAD_CONNECTION
        |
        v
mEnableNetBounce = false
        |
        v
networkBounceDisabled
        |
        v
skipResetConnection
        |
        v
Wi-Fi remains associated
        |
        v
EchoMuse remains connected
```

**Next Step:** Extract the exact Intent extra from `WifiDiagsConfigReceiver.onReceive()` and perform a controlled runtime test.

---

## Persistence

The discovered control appears to operate inside:

```text
system_server
```

No evidence currently confirms that a successful value survives a reboot.

Until verified otherwise, treat the setting as runtime-only.

If the runtime broadcast successfully disables network bounce but the value returns to `true` after reboot, persistence can be handled by issuing the verified broadcast during the existing rooted startup process.

The preferred sequence would be:

```text
Fire OS boot
        |
        v
system_server starts
        |
        v
WifiDiagsUtil initializes
        |
        v
root startup process
        |
        v
SET_ENABLENETBOUNCE false
        |
        v
EchoMuse operates normally
```

Do not implement persistence until the runtime command itself has been validated.

**Next Step:** Validate the runtime control first; test reboot persistence only after the disconnect behavior is resolved.

---

## Alternative Fixes

### Disable Wi-Fi Diagnostics

The framework exposes:

```text
SET_ENABLEWIFIDIAGS
```

and references to stopping Wi-Fi diagnostics.

This may provide another way to prevent the unwanted recovery behavior.

It is not the preferred first fix because network bounce has its own dedicated control.

### Disable Individual Connectivity Tests

The framework exposes controls related to:

```text
SET_ENABLECAPTIVEPORTALCHECK
SET_PERFORMHTTPTEST
SET_PERFORMPINGTEST
```

Disabling one or more tests could potentially prevent `BAD_CONNECTION`.

This has not been tested.

### Modify Fire OS Framework

The responsible framework is:

```text
/system/framework/arm/fosservices.odex
```

The relevant class is:

```text
com.amazon.android.service/wifidiags/WifiDiagsUtil
```

Direct modification could potentially force network bounce off.

This is a last-resort option because the framework executes inside:

```text
system_server
```

A defective framework modification could affect normal Fire OS operation.

### Fix Priority

Use this order:

```text
1. Existing SET_ENABLENETBOUNCE runtime control
2. Other existing WifiDiags runtime controls
3. Startup persistence for a validated runtime fix
4. Framework modification only if no supported runtime control works
```

**Next Step:** Do not proceed to alternative fixes until the exact `SET_ENABLENETBOUNCE` Intent extra has been investigated.

---

## Validation Procedure

### Step 1 - Confirm Baseline Association

Run:

```bash
wpa_cli -i wlan0 -p /data/misc/wifi/sockets status
```

Expected:

```text
wpa_state=COMPLETED
```

Record the current BSSID.

For this sanitized documentation, the example BSSID is:

```text
02:00:00:00:00:01
```

---

### Step 2 - Apply the Runtime Fix

After the exact Intent extra is identified:

```bash
am broadcast \
    -a com.amazon.android.service.wifidiags.SET_ENABLENETBOUNCE \
    --ez <VERIFIED_EXTRA> false
```

Expected Android result:

```text
Broadcast completed: result=0
```

This result alone does not prove the setting changed.

---

### Step 3 - Verify the Received Value

Run:

```bash
logcat -d | busybox grep -i 'receivedIntentSetEnableNetBounce' | busybox tail -n 5
```

Required result:

```text
event=receivedIntentSetEnableNetBounce, val=false
```

If the result remains:

```text
val=true
```

the fix has not been applied.

---

### Step 4 - Monitor Network Bounce

Run:

```bash
logcat -v threadtime | busybox grep -E \
'BAD_CONNECTION|networkBounce|receivedIntentSetEnableNetBounce'
```

A successful disabled state should eventually produce:

```text
event=networkBounceDisabled, action=skipResetConnection
```

The diagnostic subsystem may continue reporting:

```text
BAD_CONNECTION
```

That is acceptable if the network is no longer reset.

---

### Step 5 - Validate EchoMuse

Monitor EchoMuse through the next expected network-bounce interval.

The primary validation window should exceed:

```text
42 minutes 18 seconds
```

Use at least 45-60 minutes for the first controlled test.

The EchoMuse controller should remain connected through the expected boundary.

---

### Step 6 - Verify Wi-Fi State

After the validation interval:

```bash
wpa_cli -i wlan0 -p /data/misc/wifi/sockets status
```

Expected:

```text
wpa_state=COMPLETED
```

The Echo should remain associated without a Fire OS initiated bounce.

**Validation Result:** The fix is considered operational only after `val=false`, `networkBounceDisabled`, and uninterrupted operation through the expected bounce interval are all observed.

---

## Troubleshooting Commands

### Show Wi-Fi Association

```bash
wpa_cli -i wlan0 -p /data/misc/wifi/sockets status
```

Displays the current BSSID, SSID, frequency, security state, IP address, and WPA association state.

Sanitized expected identifiers include:

```text
bssid=02:00:00:00:00:01
ssid=iot-matter
ip_address=198.51.100.60
```

---

### Show Wireless Interface

```bash
ip addr show wlan0
```

Displays interface state and assigned IPv4 and IPv6 addresses.

The sanitized Echo Dot IPv4 address used by this document is:

```text
198.51.100.60
```

---

### Show Routes

```bash
ip route
```

Displays the IPv4 default route and connected networks.

Sanitized default route:

```text
default via 198.51.100.1 dev wlan0
```

---

### Show EchoMuse Server

```bash
ps | busybox grep /data/local/bin/server
```

Confirms whether the EchoMuse server process remains running.

---

### Show Wi-Fi Processes

```bash
ps | busybox grep -E 'wpa|wifi|dhcp'
```

Displays relevant Wi-Fi and DHCP processes.

---

### Show Fire OS Wi-Fi Properties

```bash
getprop | busybox grep -i wifi
```

Displays Wi-Fi-related Android properties.

Review output before publishing because runtime properties may contain environment-specific values.

---

### Show WLAN Properties

```bash
getprop | busybox grep -i wlan
```

Displays WLAN interface, DHCP, and MediaTek properties.

Review output before publishing because DHCP or interface properties may contain production network addressing.

---

### Review Wi-Fi Diagnostics

```bash
logcat -d -v threadtime | busybox grep 'WifiDiagsUtil'
```

Displays historical `WifiDiagsUtil` events with PID, TID, and timestamps.

Review captured output before public publication for:

* Private IPv4 addresses.
* Internal IPv6 addresses.
* SSIDs.
* BSSIDs.
* Hostnames.
* Internal DNS servers.
* Device-specific identifiers.

---

### Monitor Network Bounce

```bash
logcat -v threadtime | busybox grep -E \
'BAD_CONNECTION|networkBounce|receivedIntentSetEnableNetBounce'
```

Monitors the primary Fire OS events involved in the recurring disconnect.

Stop with:

```text
Ctrl+C
```

---

### Review MediaTek Driver Events

```bash
busybox dmesg | busybox grep -i -E \
'wlan|wifi|mtk|disconnect|deauth|scan'
```

Displays relevant kernel and MediaTek wireless events.

Review captured output before publication for BSSIDs, interface addressing, or other device-specific identifiers.

---

### Identify a Process

Example for PID `1644`:

```bash
busybox cat /proc/1644/cmdline
```

Expected for the observed `WifiDiagsUtil` process:

```text
system_server
```

The PID is an observed runtime process identifier and is not expected to remain consistent across boots.

---

### Inspect Process Mappings

```bash
busybox cat /proc/1644/maps | busybox grep -E '\.apk|\.jar|\.odex'
```

Used to identify framework components loaded into `system_server`.

---

### Extract Fire OS Framework Strings

```bash
busybox strings /system/framework/arm/fosservices.odex > /tmp/fosstrings
```

Creates a searchable string dump without modifying the framework.

---

### Search Network-Bounce Strings

```bash
busybox grep -i -E 'netbounce|networkbounce' /tmp/fosstrings
```

Displays network-bounce constants, fields, event messages, and method names.

---

### Search Intent Fields

```bash
busybox grep -i -E '^.*FIELD$|^.*_FIELD$|^.*field.*bounce.*$' /tmp/fosstrings
```

Displays discovered Intent extra field symbols.

---

### Confirm the Broadcast Receiver

```bash
am broadcast \
    -a com.amazon.android.service.wifidiags.SET_ENABLENETBOUNCE
```

Current verified result:

```text
event=receivedIntentSetEnableNetBounce, val=true
```

This confirms the receiver but does not disable network bounce.

**Validation Result:** These commands provide the required operational visibility to reproduce, diagnose, and validate the issue without modifying Fire OS.

---

## Commands and Changes to Avoid

Do not kill:

```text
wpa_supplicant
```

Do not terminate:

```text
system_server
```

Do not remove:

```text
fosservices.odex
```

Do not modify the MediaTek Wi-Fi driver.

Do not modify the wireless infrastructure solely to work around this behavior without evidence that an access point configuration is responsible.

Do not assume:

```text
Broadcast completed: result=0
```

means a configuration change succeeded.

The authoritative runtime confirmation for the planned fix is:

```text
event=receivedIntentSetEnableNetBounce, val=false
```

followed during a failed diagnostic cycle by:

```text
event=networkBounceDisabled, action=skipResetConnection
```

**Next Step:** Make no invasive changes until the existing runtime disable mechanism has been fully tested.

---

## Current Technical Assessment

The primary recurring EchoMuse controller disconnect is no longer an unexplained wireless failure.

The verified sequence is:

```text
Fire OS connectivity diagnostics
        |
        v
BAD_CONNECTION
        |
        v
Amazon network-bounce state machine
        |
        v
networkBounce
        |
        v
software-requested Wi-Fi disassociation
        |
        v
MediaTek DisconnectByOid
        |
        v
wlan0 interruption
        |
        v
Wi-Fi scan and reassociation
        |
        v
EchoMuse controller interruption
```

The following evidence supports this determination:

* Strong Echo Dot RF signal.
* Strong wireless access point signal measurements.
* A nearby device on the same SSID remaining connected.
* `wpa_state=COMPLETED` during normal operation.
* `WifiDiagsUtil` reporting `BAD_CONNECTION`.
* Explicit `networkBounce` events.
* MediaTek logging `DisconnectByOid`.
* The Echo reconnecting to the same wireless access point.
* A captured approximately 42-minute network-bounce recurrence.
* Discovery of `NETBOUNCE_*` timing constants.
* Discovery of `mEnableNetBounce`.
* Discovery of `networkBounceDisabled`.
* Discovery of `WifiDiagsConfigReceiver`.
* Discovery of the `SET_ENABLENETBOUNCE` broadcast action.
* Successful delivery of that broadcast to `WifiDiagsUtil`.

The problem appears fixable because Fire OS already contains an explicit state that suppresses the network reset:

```text
event=networkBounceDisabled, action=skipResetConnection
```

The unresolved task is determining the exact boolean Intent extra that causes:

```text
event=receivedIntentSetEnableNetBounce, val=false
```

No permanent fix should be considered validated until that state is achieved and the Echo remains connected through the expected network-bounce interval.

**Next Step:** Inspect `WifiDiagsConfigReceiver.onReceive()` to determine the exact `INTENT_EXTRA_ENABLENETBOUNCE_FIELD` literal.

---

## Related Search Keywords

amazon-echo-dot-2, echomuse, home-assistant, fire-os, android, mt8163, wpa-supplicant, wifidiagsutil, wifidiagsconfigreceiver, netbounce, fosservices, system-server, wifi-disconnect, network-bounce, mdns, local-voice-satellite, wifi-diagnostics, root-cause-analysis

---

## Revision Control

| Version | Date | Summary | Author |
| --- | --- | --- | --- |
| **1.0.0** | 2026-08-26 | Initial documentation of the Echo Dot 2 recurring EchoMuse disconnect investigation, confirmed Fire OS network-bounce cause, runtime control discovery, validation evidence, sanitization reference, and remaining remediation work. | projectfong |
