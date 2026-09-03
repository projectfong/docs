# Amazon Echo Dot 2 to EchoMuse Home Assistant Voice Satellite - Quickstart

Author: projectfong    
Copyright (c) 2026 Fong

---

## Summary

Convert an Amazon Echo Dot 2nd Generation RS03QR into a local Home Assistant voice satellite using:

```text
Amonet
TWRP
Fire OS 5
Magisk 17.3
EchoMuse
ESPHome
Home Assistant Assist
```

Final voice path:

```text
"Hey Jarvis"
      |
      v
Echo Dot 2
      |
      v
EchoMuse
      |
      v
ESPHome Voice Satellite
      |
      v
Home Assistant Assist
      |
      +--> faster-whisper
      |
      +--> Conversation Agent
      |
      +--> Piper
      |
      v
Echo Dot Speaker
```

This quickstart assumes familiarity with Linux, ADB, fastboot, Home Assistant, ESPHome, and basic network troubleshooting.

For recovery procedures and detailed troubleshooting, use the full implementation guide.

## Known-Good Baseline

| Component | Known-Good Value |
| --- | --- |
| Echo | Amazon Echo Dot 2nd Generation |
| Physical model | RS03QR |
| Platform | `biscuit` |
| Android product | `csm_biscuit` |
| Android | 5.1.1 |
| Fire OS | 5.5.5.4 |
| Fire OS build | `680767620` |
| Amonet | `amonet-biscuit-v1.1.0` |
| Magisk | 17.3 |
| EchoMuse device | v2.12.0 |
| EchoMuse Controller | v2.20.2 |
| Wake word | `hey_jarvis_v0.1` |
| STT | faster-whisper |
| TTS | Piper |

These versions represent the documented known-good deployment.

Do not assume newer versions are interchangeable without validating compatibility.

## Critical Model Warning

This procedure applies to:

```text
Amazon Echo Dot 2nd Generation
Model: RS03QR
Platform: biscuit
```

Do not use the `amonet-biscuit` procedure on another Echo model.

Confirm the physical model before continuing.

## Prepare Linux

On Debian/Ubuntu:

```bash
sudo apt update
sudo add-apt-repository universe
sudo apt install python3 python3-serial adb fastboot dos2unix
```

Disable ModemManager:

```bash
sudo systemctl stop ModemManager
sudo systemctl disable ModemManager
```

Extract:

```text
amonet-biscuit-v1.1.0.zip
```

Enter the Amonet directory:

```bash
cd ~/Downloads/amonet-biscuit-v1.1.0/amonet
```

## Unlock the Echo

### Try the Supported Entry Path First

With the Echo powered off:

```text
1. Hold the action/circle button.
2. Reconnect power.
3. Wait for the expected Amonet indication.
```

If Amonet reports:

```text
unsupported version
```

do not attempt random partition-writing commands.

Use the hardware BootROM procedure.

## Hardware BootROM Entry

The Amonet RS03QR procedure uses a PCB test point near the Samsung eMMC.

The documented point is:

```text
Immediately above the Samsung eMMC,
between the R60 and C52 board markings.
```

Important:

```text
Do not short the C52 capacitor itself.
```

Use the upstream Amonet/XDA board image to positively identify the test point before applying power.

With the device unpowered:

```text
1. Open the Echo.
2. Expose the PCB.
3. Short the documented BootROM test point to ground.
4. Start bootrom-step.sh.
5. Connect USB to the Linux host.
6. Maintain the short.
7. Release it only when the script instructs you.
```

Run:

```bash
sudo ./bootrom-step.sh
```

After entering hacked fastboot:

```bash
sudo ./fastboot-step.sh
```

Complete the prompts until TWRP is available.

## Useful Boot Commands

Fire OS to TWRP:

```bash
adb reboot recovery
```

Hacked fastboot to TWRP:

```bash
fastboot oem reboot-recovery
```

TWRP to Amonet fastboot:

```bash
adb shell reboot-amonet
```

Do not assume:

```bash
adb reboot
```

is equivalent to:

```bash
adb shell reboot-amonet
```

## Install Fire OS 5

Known-good image:

```text
update-kindle-csm_biscuit-272.6.8.0_user_680767620.bin
```

Corresponding version:

```text
Fire OS 5.5.5.4
Build 680767620
```

From TWRP, wipe:

```bash
adb shell twrp wipe data
adb shell twrp wipe cache
```

Push `f1r30s`:

```bash
adb push f1r30s.zip /sdcard/
```

Start sideload:

```bash
adb shell twrp sideload
```

Sideload Fire OS:

```bash
adb sideload update.bin
```

Install `f1r30s`:

```bash
adb shell twrp install /sdcard/f1r30s.zip
```

Boot Fire OS.

## A/B Partition Warning

The unlocked Echo uses Amonet-aware remapped boot locations including:

```text
boot_a_x
boot_b_x
```

Do not casually flash boot or recovery images from Fire OS using generic flashing tools.

Use the documented TWRP/Amonet-aware procedure.

## Verify ADB

Run:

```bash
adb devices -l
```

Expected device characteristics:

```text
product:csm_biscuit
model:AEOBC
device:biscuit
```

If Linux reports USB permissions problems:

```bash
adb kill-server
sudo adb start-server
sudo adb devices -l
```

## Install EchoMuse Controller

Add the EchoMuse repository to Home Assistant:

```text
https://github.com/wilbowes/EchoMuse
```

Install the EchoMuse Controller.

The documented controller exposes:

```text
TCP/8767 - discovery / WebSocket
TCP/8770 - device-link WSS/TLS
TCP/8768 - dashboard/API
```

EchoMuse advertises:

```text
_emcontroller._tcp.local
```

## Prepare Chrome/WebUSB

EchoMuse provisioning uses Chrome/Chromium WebUSB.

Before using WebUSB, release the Echo from the native ADB daemon:

```bash
adb kill-server
```

Do not immediately run `adb devices` again.

Return directly to Chrome and connect through EchoMuse.

### HTTP Home Assistant Management

If Chrome reports:

```text
WebUSB not available - requires a secure context
```

and Home Assistant is intentionally accessed over HTTP, Chrome can be configured to treat the exact management origin as secure:

```text
chrome://flags/#unsafely-treat-insecure-origin-as-secure
```

Add only the intended Home Assistant management origin and relaunch Chrome.

### Provisioning Workstation Internet

The EchoMuse provisioning interface may retrieve browser-side dependencies from:

```text
https://esm.sh/
```

The provisioning workstation may therefore temporarily require Internet access.

This does not mean the Echo requires unrestricted Internet access during normal operation.

## Run EchoMuse Provisioning

Connect the Echo through the EchoMuse provisioning wizard.

Expected identification:

```text
Model: AEOBC
Android: 5.1.1
Codename: csm_biscuit
Fire OS: 5.5.5.4
```

Allow the wizard to:

```text
Connect to TWRP
Patch the boot image
Configure SELinux as required
Install Magisk
Configure root authorization
Disable interfering Amazon services
Configure Wi-Fi
Install EchoMuse
Install wake-word assets
Reboot the Echo
```

## Install Magisk 17.3

Use:

```text
Magisk-v17.3.zip
```

Known-good SHA256 from the documented deployment:

```text
18e46b16b25ebe691c282fe311beccd4811cd533848a64e2efbd754fb85efde7
```

Do not substitute a current Magisk release without separately validating compatibility with this platform and EchoMuse workflow.

After boot, verify:

```bash
adb shell su -c '/sbin/magisk -v'
```

Expected:

```text
17.3:MAGISK
```

Version code:

```bash
adb shell su -c '/sbin/magisk -V'
```

Expected:

```text
17302
```

## Configure IoT Wi-Fi

Example sanitized configuration:

```text
SSID: ha-iot
IoT network: 10.20.90.0/24
Echo: 10.20.90.60
Gateway: 10.20.90.1
```

Expected `wpa_supplicant` structure:

```text
network={
        ssid="ha-iot"
        scan_ssid=1
        psk="<REDACTED>"
        key_mgmt=WPA-PSK
        priority=4
}
```

Never publish the real WPA passphrase.

## Verify EchoMuse Device Service

Check:

```bash
adb shell su -c 'ps | grep /data/local/bin/server'
```

Review:

```bash
adb shell su -c 'cat /tmp/server.log'
```

Expected startup behavior includes:

```text
EchoMuse starting
PcmSpeaker initialised
Volume controller initialised
Ready
mDNS: browsing for _emcontroller._tcp.local...
```

## Segmented Network Requirements

For a routed IoT deployment:

```text
Services Network
      |
      | Home Assistant
      | EchoMuse
      |
   Firewall
      |
      | Routed unicast
      |
IoT Network
      |
      | Echo Dot 2
```

Normal unicast traffic should route through the firewall.

mDNS/DNS-SD must be handled separately when discovery crosses VLAN boundaries.

Required EchoMuse communication includes:

```text
Echo -> EchoMuse TCP/8767
Echo -> EchoMuse TCP/8770
```

mDNS uses:

```text
UDP/5353
224.0.0.251
```

Do not broadly flatten network segmentation merely to make discovery work.

## Verify the Route First

Before troubleshooting EchoMuse discovery, verify that the Echo has a valid route to the controller.

Example:

```bash
adb shell su -c 'ip route get 10.20.30.10'
```

Expected:

```text
10.20.30.10 via 10.20.90.1 dev wlan0 src 10.20.90.60
```

If the result is:

```text
RTNETLINK answers: Network is unreachable
```

stop.

Fix Wi-Fi or routing before troubleshooting:

```text
mDNS
Avahi
EchoMuse
TLS
ESPHome
Home Assistant
```

## Verify Wi-Fi

Run:

```bash
adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 status'
```

Expected:

```text
ssid=ha-iot
wpa_state=COMPLETED
ip_address=10.20.90.60
```

If the saved network exists but the Echo remains scanning:

```bash
adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 select_network 0'
```

Then verify status again.

## Cold-Boot Wi-Fi Recovery

The documented deployment experienced a cold-boot condition where the saved network remained configured but was not automatically selected.

Check:

```bash
adb shell su -c \
'getprop init.svc.smarthomewifid'
```

If the Amazon Wi-Fi manager is running:

```bash
adb shell su -c \
'stop smarthomewifid'
```

Then:

```bash
adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 select_network 0'
```

Verify:

```text
wpa_state=COMPLETED
```

## Persistent Wi-Fi Helper

Magisk 17.3 uses:

```text
/sbin/.core/img/.core/service.d
```

The documented deployment uses a boot helper to stop `smarthomewifid` and reselect the configured network.

Create:

```sh
#!/system/bin/sh

(
    sleep 60

    stop smarthomewifid 2>/dev/null

    WPA_CLI=/system/bin/wpa_cli
    CTRL=/data/misc/wifi/sockets

    i=0
    while [ $i -lt 60 ]; do
        if [ -S "$CTRL/wlan0" ]; then
            "$WPA_CLI" -p "$CTRL" -i wlan0 select_network 0 >/dev/null 2>&1
            exit 0
        fi

        sleep 2
        i=$((i + 1))
    done
) &
```

Install it as:

```text
/sbin/.core/img/.core/service.d/echomuse-wifi.sh
```

Set permissions:

```bash
chmod 755 /sbin/.core/img/.core/service.d/echomuse-wifi.sh
```

This helper is deployment-derived recovery logic rather than a generic requirement for every EchoMuse installation.

## mDNS Across VLANs

EchoMuse advertises:

```text
_emcontroller._tcp.local
```

If the Echo and controller are on separate VLANs, provide deliberate mDNS/DNS-SD reflection or publication.

Verify:

```bash
avahi-browse -rt _emcontroller._tcp
```

Expected structure:

```text
hostname = [echomuse.local]
address = [10.20.30.10]
port = [8767]
txt = ["tls_port=8770" "server=echomuse" "version=1"]
```

If necessary, inspect IoT-side mDNS:

```bash
sudo tcpdump -ni <IOT_INTERFACE> udp port 5353
```

Expected Echo query:

```text
PTR? _emcontroller._tcp.local.
```

## Direct DNS-SD Publication

If normal reflection is unreliable, the controller service can be explicitly published on the IoT-facing Avahi environment:

```bash
avahi-publish-service \
-H echomuse.local \
echomuse-direct \
_emcontroller._tcp \
8767 \
version=1 \
server=echomuse \
tls_port=8770
```

This publishes discovery information.

It does not proxy EchoMuse unicast traffic.

The Echo still connects to the actual controller through normal routed networking.

## Approve the Echo

Once discovery works, EchoMuse may show the device as:

```text
PENDING APPROVAL
```

Approve it.

A healthy connection should progress through states equivalent to:

```text
mDNS controller found
control channel connected
device registered
data channel connected
microphone streaming started
```

## Add the ESPHome Voice Satellite

EchoMuse exposes the approved Echo as an ESPHome voice-satellite endpoint.

Home Assistant should discover a device similar to:

```text
echodot2 Voice Assistant
```

Add it through:

```text
Settings
-> Devices & services
-> ESPHome
```

This step is required.

A working wake word alone does not prove Home Assistant is connected to the voice satellite.

## Configure the Assist Pipeline

Configure the desired Home Assistant Assist pipeline.

Example local pipeline:

```text
Wake word:
OpenWakeWord

Model:
hey_jarvis_v0.1

STT:
faster-whisper

Conversation:
Home Assistant or local conversation agent

TTS:
Piper
```

## Validate Wake Word

Say:

```text
Hey Jarvis
```

Confirm EchoMuse/OpenWakeWord detects the wake word.

This validates:

```text
Echo microphones
        |
        v
EchoMuse audio
        |
        v
OpenWakeWord
```

## Validate Speech-to-Text

Speak a simple request.

Example:

```text
What's 10 plus 10?
```

Confirm faster-whisper produces the correct transcript.

A correct transcript establishes that this path is working:

```text
Echo microphone
      |
      v
EchoMuse
      |
      v
ESPHome
      |
      v
Home Assistant
      |
      v
faster-whisper
```

If transcription is correct but the request fails afterward, troubleshoot later Assist pipeline stages rather than restarting at Wi-Fi or microphone diagnostics.

## Final Validation

Confirm:

```text
[ ] Device is RS03QR / biscuit
[ ] Amonet unlock completed
[ ] TWRP available
[ ] Fire OS 5.5.5.4 installed
[ ] f1r30s installed
[ ] ADB works
[ ] Magisk 17.3 works
[ ] EchoMuse device service runs
[ ] Echo joins IoT Wi-Fi
[ ] Echo has valid default route
[ ] Echo can route to EchoMuse
[ ] EchoMuse DNS-SD is discoverable
[ ] TCP/8767 reachable as required
[ ] TCP/8770 reachable as required
[ ] Echo approved in EchoMuse
[ ] ESPHome voice satellite added to HA
[ ] Wake word detected
[ ] faster-whisper receives speech
[ ] Conversation pipeline responds
[ ] Piper returns speech
[ ] Echo speaker plays response
```

Then perform a physical power cycle.

After reboot, confirm:

```text
[ ] Wi-Fi reconnects
[ ] Correct IoT network selected
[ ] Route to EchoMuse returns
[ ] EchoMuse reconnects
[ ] ESPHome satellite returns
[ ] Wake word works
[ ] Complete voice request works
```

## Troubleshooting Order

Troubleshoot from the bottom of the stack upward:

```text
1. Device power / Android
2. Wi-Fi association
3. IP address
4. Routing
5. Firewall
6. mDNS / DNS-SD
7. EchoMuse connection
8. ESPHome connection
9. Wake word
10. STT
11. Conversation agent
12. TTS
13. Speaker output
```

Do not troubleshoot a higher layer until the lower layers are known to work.

### No Route to EchoMuse

Run:

```bash
adb shell su -c 'ip route get 10.20.30.10'
```

If unreachable, fix Wi-Fi/routing first.

### Wi-Fi Stuck Scanning

Run:

```bash
adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 status'
```

If necessary:

```bash
adb shell su -c \
'stop smarthomewifid'
```

Then:

```bash
adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 select_network 0'
```

### Echo Cannot Discover Controller

First confirm routing works.

Then verify:

```bash
avahi-browse -rt _emcontroller._tcp
```

and inspect UDP/5353 if necessary.

### Wake Word Works but Voice Turn Fails

If EchoMuse reports behavior equivalent to:

```text
no active HA connection
```

verify the EchoMuse ESPHome satellite has been added to Home Assistant.

### STT Works but No Response

If faster-whisper produces the correct transcript, the microphone, EchoMuse transport, ESPHome transport, and STT path are already functioning.

Troubleshoot:

```text
Conversation agent
Assist pipeline
TTS
Response playback
```

## Security Notes

A segmented deployment can maintain:

```text
Echo Dot -> Internet:           DENY
Echo Dot -> user LAN:           DENY
Echo Dot -> unrelated services: DENY

Echo Dot -> EchoMuse TCP/8767:  ALLOW
Echo Dot -> EchoMuse TCP/8770:  ALLOW
```

Handle mDNS deliberately rather than broadly forwarding multicast between all networks.

The provisioning workstation may temporarily require Internet access for browser-side dependencies.

Do not publish:

```text
Wi-Fi credentials
Device serial numbers
MAC addresses
BSSIDs
EchoMuse tokens
Home Assistant tokens
TLS private keys
Authentication material
```

## Full Documentation

This quickstart intentionally omits:

```text
Detailed unlock investigation
Extended BootROM troubleshooting
Full EchoMuse provisioning logs
Amazon package-removal lists
Detailed Wi-Fi investigation
p2p0 investigation
Extended Avahi diagnostics
Packet-capture examples
Full diagnostic command reference
Historical failure analysis
Recovery workflow
Detailed component explanations
```

See the full Amazon Echo Dot 2 to EchoMuse to Home Assistant OS documentation for those procedures and diagnostic details.

## Credits and Upstream Projects

The low-level Echo Dot 2 unlock and recovery procedures are based on the work of the Amonet and XDA Developers community.

EchoMuse provides the device-side software, controller, provisioning workflow, and Home Assistant voice-satellite functionality used by this implementation.

Refer to the upstream Amonet/XDA and EchoMuse projects for current source code, project-specific instructions, authorship, and compatibility information.

## Related Search Keywords

amazon-echo-dot-2, rs03qr, echomuse, home-assistant, esphome, voice-assistant, amonet, bootrom, twrp, magisk, fireos-5, openwakeword, faster-whisper, piper, avahi, mdns, iot-security, network-segmentation

## Revision Control

| Version | Date | Summary | Author |
| --- | --- | --- | --- |
| **1.0.0** | 2026-08-31 | Initial Amazon Echo Dot 2 EchoMuse Home Assistant voice satellite quickstart. | projectfong |
