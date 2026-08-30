# Amazon Echo Dot 2 to EchoMuse to Home Assistant OS

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

This guide documents a reproducible process for converting an Amazon Echo Dot 2nd Generation RS03QR into a local Home Assistant voice satellite using Amonet, TWRP, Fire OS 5, Magisk 17.3, EchoMuse, ESPHome, and the Home Assistant Assist pipeline. It also covers the network, mDNS, Wi-Fi persistence, and troubleshooting work required when the Echo Dot and Home Assistant are separated across routed VLANs.

This public version intentionally removes deployment-specific IP addresses, device serial numbers, MAC addresses, Wi-Fi credentials, authentication material, and other environment-specific identifiers.

---

## Scope

This guide covers only components that directly affect using an Echo Dot 2 as a Home Assistant voice satellite.

Included:

* Amazon Echo Dot 2 RS03QR hardware identification
* Amonet `biscuit` unlock
* unsupported-firmware BootROM recovery path
* hardware test-point method
* hacked fastboot
* TWRP
* Fire OS 5.5.5.4
* `f1r30s`
* ADB
* EchoMuse Controller
* Chrome and WebUSB
* Magisk 17.3
* EchoMuse provisioning
* Alexa/Amazon service removal
* IoT Wi-Fi configuration
* EchoMuse device service
* wake-word assets
* routed VLAN architecture
* Avahi/mDNS
* direct EchoMuse DNS-SD publication
* Wi-Fi recovery
* `smarthomewifid`
* Magisk boot-time service scripts
* ESPHome voice satellite integration
* OpenWakeWord
* faster-whisper
* Piper
* Home Assistant Assist
* diagnostic and recovery procedures

Excluded:

* unrelated Matter devices
* smart plugs and switches
* dashboard design
* unrelated ESP32 projects
* garage-door automation
* unrelated homelab configuration

Validation Result: The documented process results in an Echo Dot 2 operating as a locally controlled Home Assistant ESPHome voice satellite.

---

## Example Network Used in This Guide

All addresses below are fictional examples.

They are deliberately different from the original deployment.

### Network Layout

| Network                   | Example Subnet  | Purpose                        |
| ------------------------- | --------------- | ------------------------------ |
| Services / Home Assistant | `10.20.30.0/24` | HAOS, EchoMuse, infrastructure |
| IoT                       | `10.20.90.0/24` | Echo Dot and IoT devices       |

Example hosts:

| Component                  | Example Address |
| -------------------------- | --------------- |
| HAOS / EchoMuse Controller | `10.20.30.10`   |
| Avahi services interface   | `10.20.30.20`   |
| Avahi IoT interface        | `10.20.90.20`   |
| Echo Dot 2                 | `10.20.90.60`   |
| IoT gateway                | `10.20.90.1`    |

These addresses form a valid routed RFC1918 topology and remain consistent throughout this document.

---

## Final Architecture

```text
                         Services Network
                         10.20.30.0/24
                                |
                  +-------------+-------------+
                  |                           |
           HAOS / EchoMuse               vm-avahi
             10.20.30.10                10.20.30.20
                  |                           |
                  |                           |
                  +--------- Firewall --------+
                                |
                                |
                            IoT Network
                         10.20.90.0/24
                                |
                  +-------------+-------------+
                  |                           |
              vm-avahi                   Echo Dot 2
            10.20.90.20                 10.20.90.60
                                             |
                                             |
                                         EchoMuse
```

The Avahi system assists with mDNS/DNS-SD discovery.

It is not the router between the IoT and services networks.

Normal unicast traffic remains routed through the firewall.

---

## Relevant Ports

| Purpose                        | Protocol/Port |
| ------------------------------ | ------------- |
| mDNS                           | UDP 5353      |
| Home Assistant frontend        | TCP 8123      |
| EchoMuse discovery / WebSocket | TCP 8767      |
| EchoMuse device-link WSS/TLS   | TCP 8770      |
| EchoMuse ESPHome satellite 1   | TCP 16001     |
| EchoMuse dashboard/API         | TCP 8768      |

---

## EchoMuse DNS-SD Service

EchoMuse advertises:

```text
_emcontroller._tcp.local
```

Example known-good advertisement:

```text
instance: echomuse
hostname: echomuse.local
address: 10.20.30.10
port: 8767

TXT:
version=1
server=echomuse
tls_port=8770
```

Validation Result: mDNS identifies the controller, while routed unicast communication occurs through the firewall.

---

## Voice Processing Architecture

```text
"Hey Jarvis"
      |
      v
Echo Dot 2 microphone array
      |
      v
EchoMuse device service
      |
      v
EchoMuse Controller
      |
      +--> OpenWakeWord
      |
      v
ESPHome voice satellite
      |
      v
Home Assistant Assist
      |
      +--> faster-whisper
      |
      +--> conversation agent
      |
      +--> Piper
      |
      v
EchoMuse
      |
      v
Echo Dot speaker
```

---

## Hardware and Software Baseline

| Component           | Known-Good Value               |
| ------------------- | ------------------------------ |
| Echo hardware       | Amazon Echo Dot 2nd Generation |
| Physical model      | RS03QR                         |
| Platform            | `biscuit`                      |
| Android product     | `csm_biscuit`                  |
| Reported model      | `AEOBC`                        |
| Android             | 5.1.1                          |
| Fire OS             | 5.5.5.4                        |
| Fire OS build       | `680767620`                    |
| Fire OS package     | `272.6.8.0_user_680767620`     |
| Amonet              | `amonet-biscuit-v1.1.0`        |
| Magisk              | 17.3                           |
| Magisk version code | `17302`                        |
| EchoMuse device     | v2.12.0                        |
| EchoMuse Controller | v2.20.2                        |
| Home Assistant      | HAOS                           |
| Wake word           | `hey_jarvis_v0.1`              |
| STT                 | faster-whisper                 |
| TTS                 | Piper                          |

Next Step: Confirm the physical unit is an RS03QR before proceeding.

---

## Part 1 - Prepare the Echo Dot

### Verify Model Compatibility

This procedure applies to:

```text
Amazon Echo Dot 2nd Generation
Model: RS03QR
Platform: biscuit
```

Do not use the `amonet-biscuit` procedure on another Echo model.

### Cause

Amonet depends on model-specific partition layouts, bootloader behavior, and MediaTek exploit handling.

### Fix

Confirm the model before opening or modifying the device.

---

## Part 2 - Prepare Linux for Amonet

Install the required packages on a Debian/Ubuntu-based environment:

```bash
sudo apt update
sudo add-apt-repository universe
sudo apt install python3 python3-serial adb fastboot dos2unix
```

Stop ModemManager:

```bash
sudo systemctl stop ModemManager
sudo systemctl disable ModemManager
```

### Cause

ModemManager can claim low-level USB interfaces and interfere with BootROM communication.

### Fix

Disable it before starting the Amonet process.

### Extract Amonet

The release used for this process was:

```text
amonet-biscuit-v1.1.0.zip
```

Extract it and enter the Amonet working directory.

Example:

```bash
cd ~/Downloads/amonet-biscuit-v1.1.0/amonet
```

Validation Result: The Linux environment is ready to communicate with the Echo at the bootloader level.

---

## Part 3 - Determine the Required Amonet Entry Path

A supported firmware may allow entry into fastboot by:

1. Disconnecting power.
2. Holding the action/circle button.
3. Reconnecting the device.
4. Waiting for the expected green LED indication.

During the original deployment, the software path instead reported an unsupported-version condition.

Example:

```text
unsupported version: <FIRMWARE_IDENTIFIER>
```

### Cause

The installed firmware or preloader revision was not supported by the normal software-assisted path.

### Fix

Use the documented hardware BootROM short method.

Do not continue using random `brick.sh` or partition-writing commands against an unsupported version.

---

## Part 4 - Hardware BootROM Entry

### Test Point

The Amonet RS03QR guide identifies a small exposed test point on the Echo Dot 2 PCB near the Samsung eMMC package.

The referenced target is located:

```text
Immediately above the Samsung eMMC,
between the R60 and C52 board markings.
```

Important:

```text
Do not short the C52 capacitor itself.
```

C52 is a nearby SMD component.

Use the exact Amonet RS03QR board image to verify orientation before applying power.

A fine conductive tweezer or similarly controlled tool is preferable to a large screwdriver.

### BootROM Procedure

1. Open the Echo Dot 2.
2. Expose the PCB.
3. Leave the device unpowered.
4. Connect Micro-USB to the Echo but leave the host side disconnected if convenient.
5. Short the documented BootROM test point to nearby ground.
6. Start:

```bash
sudo ./bootrom-step.sh
```

7. Connect the USB cable to the Linux computer.
8. Continue holding the short.
9. Release the short only when the script explicitly instructs you to do so.

### Cause

The short forces the MediaTek SoC into BootROM communication before the normal preloader takes control.

### Fix

Follow the script timing exactly.

Validation Result: Amonet should transition the device toward hacked fastboot.

---

## Part 5 - Complete Amonet Unlock

When the device enters hacked fastboot, the LED ring should display the Amonet fastboot indication.

Run:

```bash
sudo ./fastboot-step.sh
```

Follow the prompts.

A successful unlock transitions the Echo into TWRP.

Typical TWRP visual indication:

```text
Blinking or pulsating cyan LED
```

---

## Part 6 - TWRP Commands

From a running unlocked Fire OS system:

```bash
adb reboot recovery
```

From hacked fastboot:

```bash
fastboot oem reboot-recovery
```

From TWRP back into Amonet fastboot:

```bash
adb shell reboot-amonet
```

Important:

```text
Do not assume normal "adb reboot" is equivalent to reboot-amonet.
```

---

## Part 7 - Install Fire OS 5

The known-good Fire OS image used was:

```text
update-kindle-csm_biscuit-272.6.8.0_user_680767620.bin
```

This corresponds to:

```text
Fire OS 5.5.5.4
Build 680767620
```

The Amonet procedure used here requires Fire OS 5 after the persistent exploit is installed.

Do not treat Fire OS 6 as the normal post-unlock operating system for this configuration.

### Wipe Existing Data

From TWRP:

```bash
adb shell twrp wipe data
adb shell twrp wipe cache
```

### Push f1r30s

```bash
adb push f1r30s.zip /sdcard/
```

### Sideload Fire OS

Start sideload mode:

```bash
adb shell twrp sideload
```

Then:

```bash
adb sideload update.bin
```

Where `update.bin` is the selected Fire OS 5 image.

### Install f1r30s

After the Fire OS image completes:

```bash
adb shell twrp install /sdcard/f1r30s.zip
```

The `f1r30s` package provides functionality required by this unlocked configuration, including ADB access and protections against normal OTA behavior.

Validation Result: Fire OS 5 boots with persistent ADB functionality.

---

## Part 8 - Important A/B Partition Warning

The Echo uses an A/B-style boot arrangement.

With the exploit installed, remapped boot locations include:

```text
boot_a_x
boot_b_x
```

Do not casually flash boot or recovery images from within Fire OS using tools such as Magisk Manager or FlashFire.

Use TWRP or Amonet-aware tools.

Repeated failed boot attempts with both slots unbootable can eventually cause additional recovery complications.

---

## Part 9 - Verify ADB

Connect the Echo with a known-good USB data cable.

Run:

```bash
adb devices -l
```

Expected structure:

```text
List of devices attached
<DEVICE_SERIAL> device usb:<BUS> product:csm_biscuit model:AEOBC device:biscuit
```

Relevant values:

```text
Product: csm_biscuit
Model:   AEOBC
Device:  biscuit
```

Do not publish the physical device serial number unless required.

### Linux ADB Permission Recovery

If Linux reports insufficient permissions:

```bash
adb kill-server
sudo adb start-server
sudo adb devices -l
```

---

## Part 10 - Install EchoMuse Controller on HAOS

Use the EchoMuse repository:

```text
https://github.com/wilbowes/EchoMuse
```

In Home Assistant, navigate to the add-on/app repository management interface and add the EchoMuse repository.

Install EchoMuse.

A representative HAOS VM allocation for a voice-heavy deployment is:

```text
vCPU: 6
RAM:  8 GB
```

This is an example deployment size, not an EchoMuse requirement.

---

## Part 11 - Verify EchoMuse Controller

A known-good controller should expose functionality equivalent to:

```text
EchoMuse Controller
WebSocket: 8767
Device-link WSS/TLS: 8770
Dashboard/API: 8768
```

Example address:

```text
10.20.30.10
```

Expected DNS-SD service:

```text
echomuse._emcontroller._tcp.local
```

Example:

```text
echomuse.local
10.20.30.10
8767
tls_port=8770
```

---

## Part 12 - Prepare Chrome for WebUSB

EchoMuse provisioning uses Chrome/Chromium WebUSB.

The browser communicates directly with the Echo over USB.

```text
Echo Dot
   |
   | USB
   v
Chrome / WebUSB
   |
   v
EchoMuse provisioning interface
```

### Secure Context Error

If Chrome reports:

```text
WebUSB not available - requires a secure context
```

and Home Assistant is intentionally being accessed over HTTP, open:

```text
chrome://flags/#unsafely-treat-insecure-origin-as-secure
```

Add the exact Home Assistant origin.

Public example:

```text
http://10.20.30.10:8123
```

Relaunch Chrome.

This browser override should be limited to the intended management origin.

---

## Part 13 - Provisioning Browser Internet Requirement

The EchoMuse provisioning page may dynamically load an ADB/WebUSB JavaScript module from:

```text
https://esm.sh/
```

If the browser cannot reach it, provisioning can fail even though the Echo itself does not require Internet access.

### Cause

The provisioning browser needs to retrieve the JavaScript dependency.

### Fix

Temporarily allow Internet access for the provisioning workstation.

This does not mean the Echo requires permanent Internet access.

---

## Part 14 - Release USB from Native ADB

If:

```bash
adb devices -l
```

can see the Echo but Chrome cannot claim it, the host ADB daemon may own the USB interface.

Run:

```bash
adb kill-server
```

Do not immediately run `adb devices` again.

Return directly to Chrome and reconnect using EchoMuse.

```text
Before:
Echo -> native ADB daemon

During provisioning:
Echo -> Chrome/WebUSB -> EchoMuse
```

Validation Result: Chrome should be able to authenticate ADB through WebUSB.

---

## Part 15 - EchoMuse Provisioning

The EchoMuse wizard used during the deployment contained 13 stages.

### Step 1 - Connect Device

Expected identification:

```text
Model: AEOBC
Android: 5.1.1
Codename: csm_biscuit
Fire OS: 5.5.5.4
```

The wizard then reboots into TWRP.

### Step 2 - Connect to TWRP

Expected TWRP banner:

```text
omni_biscuit
```

### Step 3 - Patch Boot Image

EchoMuse:

* extracts `magiskboot`
* identifies the boot partition
* pulls the boot image
* adds SELinux permissive configuration
* patches the ramdisk
* repacks the image
* flashes the patched image
* verifies the result

The original boot command line was extended with:

```text
androidboot.selinux=permissive
```

The ramdisk configuration modified included:

```text
init.csm.project.rc
```

---

## Part 16 - Install Magisk 17.3

Use:

```text
Magisk-v17.3.zip
```

Do not substitute a current Magisk release without separately validating compatibility.

The wizard hashes the file before installation.

A known-good hash from the captured installation was:

```text
SHA256:
18e46b16b25ebe691c282fe311beccd4811cd533848a64e2efbd754fb85efde7
```

The installer identifies:

```text
Magisk v17.3
Device platform: arm64
```

Validation Result: Magisk installs successfully through TWRP.

---

## Part 17 - Root Database and Root Verification

EchoMuse pre-seeds the Magisk root authorization database.

After reboot, the wizard tests:

```bash
su -c id
```

Known-good result format:

```text
uid=0(root) gid=0(root) context=u:r:magisk:s0
```

Magisk may require time after Android boot before `magiskd` responds.

Do not assume failure immediately if the first root attempt does not succeed.

### Manual Magisk Verification

```bash
sudo adb shell su -c '/sbin/magisk -v'
```

Expected:

```text
17.3:MAGISK
```

Version code:

```bash
sudo adb shell su -c '/sbin/magisk -V'
```

Expected:

```text
17302
```

---

## Part 18 - Disable Amazon Voice Services

EchoMuse disables Amazon services that interfere with the local voice-satellite role.

Packages disabled during the captured provisioning process included:

```text
amazon.speech.davs.davcservice
amazon.speech.sim
com.amazon.alexa.beaconbroadcaster
com.amazon.alexa.externalmediaplayer.fireos
com.amazon.wha.mediabrowserservice
com.amazon.whisperjoin.middleware
com.amazon.whisperjoin.wss.wifiprovisioner
com.amazon.device.smarthome.dshs.services
com.amazon.mediaplayeragent
com.amazon.android.service.wifiprofilemanager
com.amazon.device.smarthome.adapters.wifi
```

EchoMuse also set:

```text
persist.wifi.migrate.complete=0
```

and stopped:

```text
smarthomewifid
```

---

## Part 19 - EchoMuse Debloat

EchoMuse additionally hides Amazon packages that are unnecessary for the new role.

Examples include:

```text
amazon.speech.sim
com.amazon.alexa.beaconbroadcaster
com.amazon.device.crashmanager
com.amazon.device.messaging
com.amazon.device.smarthome.adapters.ble
com.amazon.device.smarthome.adapters.echo
com.amazon.device.smarthome.ota
com.amazon.device.software.ota
com.amazon.echo.csm.oobe
com.amazon.mediaplayeragent
com.amazon.whad
com.amazon.whisperjoin.middleware
```

The exact provisioning build handled this automatically.

---

## Part 20 - Configure IoT Wi-Fi

Public example:

```text
SSID: ha-iot
Network: 10.20.90.0/24
Gateway: 10.20.90.1
Echo reservation: 10.20.90.60
```

Do not publish the real WPA passphrase.

The resulting `wpa_supplicant` network block should resemble:

```text
network={
        ssid="ha-iot"
        scan_ssid=1
        psk="<REDACTED>"
        key_mgmt=WPA-PSK
        priority=4
}
```

The provisioning wizard may set:

```text
scan_ssid=1
```

if the SSID is not present in its most recent scan.

This setting does not inherently prevent the network from working.

---

## Part 21 - Install EchoMuse Device Service

The provisioning wizard installs the EchoMuse binary under:

```text
/data/local/bin/
```

It also installs the startup script and device-link authentication material.

Sensitive files include:

```text
ca.pem
token
```

Do not publish device authentication tokens.

---

## Part 22 - Install Wake-Word Assets

The captured provisioning process installed:

```text
libonnxruntime.so
melspectrogram.onnx
embedding_model.onnx
hey_jarvis_v0.1.onnx
```

The selected wake-word model was:

```text
hey_jarvis_v0.1
```

The device then reboots.

Validation Result: The one-time USB provisioning phase is complete.

---

## Part 23 - Verify EchoMuse on the Device

Check the device process:

```bash
adb shell su -c 'ps | grep /data/local/bin/server'
```

Inspect the log:

```bash
adb shell su -c 'cat /tmp/server.log'
```

Expected startup structure:

```text
EchoMuse starting
PcmSpeaker initialised
Volume controller initialised
[aec] enabled
Ready
mDNS: browsing for _emcontroller._tcp.local...
```

---

## Part 24 - Verify mDNS Across VLANs

The Echo and EchoMuse Controller in this example are separated:

```text
Echo:
10.20.90.60

EchoMuse:
10.20.30.10
```

mDNS uses:

```text
224.0.0.251:5353
```

and normally remains link-local.

An Avahi reflector is therefore used between the two VLANs.

### Verify Controller Advertisement

On the Avahi host:

```bash
avahi-browse -rt _emcontroller._tcp
```

Expected:

```text
hostname = [echomuse.local]
address = [10.20.30.10]
port = [8767]
txt = ["tls_port=8770" "server=echomuse" "version=1"]
```

### Capture mDNS from the Echo

On the Avahi IoT-facing interface:

```bash
sudo tcpdump -ni <IOT_INTERFACE> udp port 5353
```

Expected Echo query:

```text
10.20.90.60.5353 > 224.0.0.251.5353:
PTR? _emcontroller._tcp.local.
```

Expected reflected response:

```text
10.20.90.20.5353 > 224.0.0.251.5353:
PTR echomuse._emcontroller._tcp.local.
```

Validation Result:

```text
Echo sends discovery query       PASS
Avahi receives query             PASS
Avahi reflects controller data   PASS
```

---

## Part 25 - Verify EchoMuse Controller Ports

From a host with permission to reach the services network:

```bash
curl 10.20.30.10:8123
```

Expected:

```text
Home Assistant HTML
```

Test WebSocket listener:

```bash
curl 10.20.30.10:8767
```

A response complaining about an invalid or missing WebSocket handshake is acceptable and confirms the listener is reachable.

Example:

```text
Failed to open a WebSocket connection
```

Test TLS/WSS port:

```bash
curl 10.20.30.10:8770
```

An empty or non-HTTP response is expected because this is not a normal HTTP endpoint.

---

## Part 26 - Critical Route Verification

This diagnostic was one of the most important findings in the original deployment.

Run on the Echo:

```bash
adb shell su -c 'ip route get 10.20.30.10'
```

Known-good result:

```text
10.20.30.10 via 10.20.90.1 dev wlan0 src 10.20.90.60
```

Critical failure:

```text
RTNETLINK answers: Network is unreachable
```

If this appears, stop troubleshooting EchoMuse, TLS, or mDNS.

The device does not currently have a usable route to the controller.

---

## Part 27 - Diagnose Echo Wi-Fi

Inspect the interface:

```bash
adb shell su -c 'ip addr show wlan0'
```

Inspect routes:

```bash
adb shell su -c 'ip route'
```

Inspect Wi-Fi state:

```bash
adb shell dumpsys wifi
```

A broken state may include:

```text
wlan0: NO-CARRIER
state DOWN

wpa_state=SCANNING
```

This means the Echo has fallen off Wi-Fi even if its saved network configuration still exists.

---

## Part 28 - Verify wpa_supplicant

Process:

```bash
adb shell su -c 'ps | grep wpa_supplicant'
```

Configuration:

```bash
adb shell su -c 'cat /data/misc/wifi/wpa_supplicant.conf'
```

Do not publish:

* WPA passphrases
* private SSIDs if they expose organizational information
* device-specific metadata unless needed

---

## Part 29 - Use the Correct wpa_cli Control Socket

Use:

```bash
wpa_cli \
-p /data/misc/wifi/sockets \
-i wlan0 \
<command>
```

Status:

```bash
adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 status'
```

Saved networks:

```bash
adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 list_networks'
```

Example:

```text
network id / ssid / bssid / flags
0       ha-iot  any
```

Scan:

```bash
adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 scan_results'
```

Do not publish the AP BSSID unless necessary.

---

## Part 30 - Force Wi-Fi Association

If the configured network is ID `0`:

```bash
sudo adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 select_network 0'
```

Expected:

```text
OK
```

Then:

```bash
sudo adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 status'
```

Expected:

```text
ssid=ha-iot
wpa_state=COMPLETED
ip_address=10.20.90.60
```

This proves:

* the PSK works
* WPA2 association works
* RF connectivity works
* DHCP works
* the failure was network selection/state rather than authentication

---

## Part 31 - Verify Routing Again

```bash
adb shell su -c 'ip route'
```

Expected:

```text
default via 10.20.90.1 dev wlan0
10.20.90.0/24 dev wlan0 proto kernel scope link src 10.20.90.60
```

Then:

```bash
adb shell su -c 'ip route get 10.20.30.10'
```

Expected:

```text
10.20.30.10 via 10.20.90.1 dev wlan0 src 10.20.90.60
```

Validation Result: The Echo has a valid unicast path to EchoMuse.

---

## Part 32 - Wi-Fi Direct Interface

Check:

```bash
adb shell su -c 'ip link show p2p0'
```

If necessary:

```bash
adb shell su -c 'ip link set p2p0 down'
```

The Wi-Fi Direct interface can complicate multicast interface selection.

However, if:

```text
ip route get <CONTROLLER_IP>
```

already fails, fix the primary Wi-Fi route before spending time on `p2p0`.

---

## Part 33 - Direct EchoMuse mDNS Publication

If reflected mDNS proves unreliable, the controller service can be directly published onto the IoT-facing Avahi environment.

Example:

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

The service still advertises:

```text
echomuse.local -> 10.20.30.10
```

The Avahi host is publishing discovery information, not acting as the destination for EchoMuse unicast traffic.

---

## Part 34 - Persistent mDNS Publication

Create:

```text
/etc/systemd/system/echomuse-mdns.service
```

with:

```ini
[Unit]
Description=Publish EchoMuse controller on mDNS
After=avahi-daemon.service network-online.target
Requires=avahi-daemon.service

[Service]
Type=simple
ExecStart=/usr/bin/avahi-publish-service -H echomuse.local echomuse-direct _emcontroller._tcp 8767 version=1 server=echomuse tls_port=8770
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Activate:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now echomuse-mdns.service
```

Verify:

```bash
systemctl status echomuse-mdns.service --no-pager
```

Expected:

```text
Active: active (running)
```

---

## Part 35 - Approve the Echo in EchoMuse

After discovery succeeds, the EchoMuse dashboard may display the device as:

```text
PENDING APPROVAL
```

Approve the device.

Do not expose the real device serial or authentication token in public screenshots.

A healthy post-approval connection should progress through states equivalent to:

```text
mDNS controller found
control channel connecting to wss://10.20.30.10:8770
device registered
data channel connecting
device identified
microphone streaming started
```

Validation Result: The Dot is now an active EchoMuse endpoint.

---

## Part 36 - Cold-Boot Wi-Fi Problem

A physical power-cycle can expose a state where the saved network still exists but the Echo remains:

```text
wpa_state=SCANNING
```

while:

```text
network 0 = ha-iot
```

still exists.

Running:

```bash
sudo adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 select_network 0'
```

can immediately restore association.

This demonstrates that persistence of the configuration and automatic selection of the configuration are separate issues.

---

## Part 37 - smarthomewifid

Check the Amazon Wi-Fi management service:

```bash
sudo adb shell su -c \
'getprop init.svc.smarthomewifid'
```

If:

```text
running
```

stop it:

```bash
sudo adb shell su -c \
'stop smarthomewifid'
```

Then reselect the network:

```bash
sudo adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 select_network 0'
```

### Cause

The remaining Amazon Wi-Fi management service was observed interfering with the intended EchoMuse Wi-Fi state.

### Fix

Stop the service and explicitly select the EchoMuse IoT network.

---

## Part 38 - Magisk 17.3 service.d Location

Magisk 17.3 uses an older layout.

The relevant service directory is:

```text
/sbin/.core/img/.core/service.d
```

Do not assume the modern path:

```text
/data/adb/service.d
```

applies to this build.

Verify:

```bash
sudo adb shell su -c \
'ls -la /sbin/.core/img/.core/service.d'
```

---

## Part 39 - EchoMuse Debloat Script

EchoMuse installs:

```text
/sbin/.core/img/.core/service.d/echomuse-debloat.sh
```

The captured script handled multiple Amazon services, but `smarthomewifid` required separate handling during troubleshooting.

---

## Part 40 - Persistent Wi-Fi Helper

A boot helper can ensure the expected Wi-Fi state is restored.

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

Purpose:

```text
Boot
 |
 v
Wait for Android
 |
 v
Stop Amazon Wi-Fi manager
 |
 v
Wait for wpa_supplicant socket
 |
 v
Select configured network
 |
 v
Restore EchoMuse connectivity
```

### Install the Helper

Create locally:

```bash
cat > /tmp/echomuse-wifi.sh <<'EOF'
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
EOF
```

Push:

```bash
adb push /tmp/echomuse-wifi.sh /sdcard/echomuse-wifi.sh
```

Install:

```bash
sudo adb shell su -c \
'cp /sdcard/echomuse-wifi.sh /sbin/.core/img/.core/service.d/echomuse-wifi.sh'
```

Permissions:

```bash
sudo adb shell su -c \
'chmod 755 /sbin/.core/img/.core/service.d/echomuse-wifi.sh'
```

Verify:

```bash
sudo adb shell su -c \
'ls -l /sbin/.core/img/.core/service.d/echomuse-wifi.sh'
```

Validation Result: Network recovery runs automatically as part of Magisk boot services.

---

## Part 41 - ESPHome Voice Satellite

EchoMuse exposes each approved Echo as an ESPHome voice-satellite endpoint.

Example first port:

```text
16001
```

Home Assistant should discover a device with a name similar to:

```text
echodot2 Voice Assistant
```

Add it under:

```text
Settings
-> Devices & services
-> ESPHome
```

This step is required.

---

## Part 42 - Wake-Word Validation

EchoMuse loads:

```text
hey_jarvis_v0.1
```

A successful wake-word engine startup should indicate that OpenWakeWord is running and has loaded the model.

A valid detection should include a score above the configured threshold.

This proves:

```text
Echo microphones      PASS
audio transport       PASS
OpenWakeWord          PASS
wake-word model       PASS
```

---

## Part 43 - Voice-Turn Failure Before ESPHome Is Added

A possible failure state is:

```text
Wake word detected
Voice turn starting
no active HA connection
cannot start voice turn
```

### Cause

Home Assistant has not connected to the EchoMuse ESPHome endpoint.

### Fix

Add the discovered ESPHome voice-satellite device.

Validation Result: Wake-word detection can work even when the HA voice turn itself cannot yet start.

---

## Part 44 - faster-whisper Validation

Configure a Home Assistant Assist pipeline using:

```text
Speech-to-text:
faster-whisper
```

Test phrases such as:

```text
What's 10 plus 10?
```

or:

```text
Tell me a short joke.
```

If faster-whisper receives the correct transcript, the following path is working:

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

At that point, do not restart troubleshooting at the microphone or mDNS layers unless new evidence indicates a regression.

---

## Part 45 - Example Local AI Pipeline

An example local voice pipeline can use:

```text
STT:
faster-whisper

Conversation agent:
local LLM through Home Assistant

TTS:
Piper
```

The original deployment separately validated that the selected local LLM backend worked through Home Assistant.

If typed Assist succeeds while Echo-originated requests fail after STT, inspect the Home Assistant voice pipeline after `stt-end`.

---

## Security and Isolation Notes

A recommended firewall model is:

```text
Echo Dot -> Internet              DENY
Echo Dot -> user LAN              DENY
Echo Dot -> unrelated services    DENY

Echo Dot -> EchoMuse TCP 8767     ALLOW
Echo Dot -> EchoMuse TCP 8770     ALLOW
```

Example:

```text
Source:
10.20.90.60

Destination:
10.20.30.10

Services:
TCP/8767
TCP/8770
```

mDNS should be handled deliberately rather than broadly forwarding multicast between every VLAN.

The Dot does not require unrestricted Internet access during normal local operation.

The provisioning workstation may temporarily require Internet access for browser-side dependencies.

Authentication files such as the EchoMuse token must never be published.

---

## Public-Safe Data Handling

Do not include the following in public documentation, screenshots, logs, or issue reports unless intentionally sanitized:

```text
real internal IP addresses
real VLAN IDs if sensitive
Wi-Fi passwords
private SSIDs if identifying
device serial numbers
MAC addresses
BSSIDs
EchoMuse authentication tokens
API keys
Home Assistant access tokens
browser event tokens
TLS private keys
personally identifying hostnames
firewall object names revealing internal structure
```

Replace values consistently.

Good public-safe example:

```text
Services VLAN:
10.20.30.0/24

IoT VLAN:
10.20.90.0/24

EchoMuse:
10.20.30.10

Echo Dot:
10.20.90.60

Gateway:
10.20.90.1
```

Bad sanitization:

```text
EchoMuse: 10.0.0.1
Echo Dot: 192.168.1.10
Gateway: 172.16.5.1
```

when the document implies those hosts are members of the same routed topology without explaining the networks.

Sanitized examples must still make architectural sense.

---

## Diagnostic Command Reference

### ADB

```bash
sudo adb devices -l
```

### Wi-Fi Status

```bash
sudo adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 status'
```

Expected:

```text
ssid=ha-iot
wpa_state=COMPLETED
ip_address=10.20.90.60
```

### Saved Networks

```bash
sudo adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 list_networks'
```

### Scan Results

```bash
sudo adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 scan_results'
```

Redact BSSIDs before publishing.

### Select Network

```bash
sudo adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 select_network 0'
```

### wlan0

```bash
sudo adb shell su -c \
'ip addr show wlan0'
```

### Routes

```bash
sudo adb shell su -c \
'ip route'
```

### Route to EchoMuse

```bash
sudo adb shell su -c \
'ip route get 10.20.30.10'
```

Expected:

```text
10.20.30.10 via 10.20.90.1 dev wlan0 src 10.20.90.60
```

### smarthomewifid

```bash
sudo adb shell su -c \
'getprop init.svc.smarthomewifid'
```

Stop:

```bash
sudo adb shell su -c \
'stop smarthomewifid'
```

### EchoMuse Process

```bash
sudo adb shell su -c \
'ps | grep /data/local/bin/server'
```

### EchoMuse Log

```bash
sudo adb shell su -c \
'cat /tmp/server.log' | tail -60
```

Review output before publishing it.

### Magisk Version

```bash
sudo adb shell su -c \
'/sbin/magisk -v'
```

```bash
sudo adb shell su -c \
'/sbin/magisk -V'
```

Expected:

```text
17.3:MAGISK
17302
```

### Magisk Services

```bash
sudo adb shell su -c \
'ls -la /sbin/.core/img/.core/service.d'
```

### DNS-SD

```bash
avahi-browse -rt _emcontroller._tcp
```

Expected public-example result:

```text
hostname = [echomuse.local]
address = [10.20.30.10]
port = [8767]
txt = ["tls_port=8770" "server=echomuse" "version=1"]
```

### Avahi

```bash
systemctl status avahi-daemon --no-pager
```

### Direct EchoMuse Publisher

```bash
systemctl status echomuse-mdns.service --no-pager
```

### mDNS Capture

```bash
sudo tcpdump -ni <IOT_INTERFACE> -vvv -s0 \
'host 10.20.90.60 and udp port 5353'
```

Expected:

```text
PTR? _emcontroller._tcp.local.
```

### Full EchoMuse Network Capture

```bash
sudo tcpdump -ni <IOT_INTERFACE> -vvv -s0 \
'host 10.20.90.60 and (udp port 5353 or tcp port 8767 or tcp port 8770)'
```

This can distinguish:

```text
mDNS failure
routing failure
firewall failure
TCP listener failure
TLS/application failure
```

---

## Troubleshooting Matrix

| Symptom                                              | Likely Cause                                    | Corrective Action                                          |
| ---------------------------------------------------- | ----------------------------------------------- | ---------------------------------------------------------- |
| Amonet reports unsupported version                   | Firmware/preloader not supported by normal path | Use documented RS03QR BootROM short                        |
| Chrome cannot claim Echo but ADB can                 | Native ADB owns USB interface                   | `adb kill-server`                                          |
| WebUSB requires secure context                       | HA accessed over HTTP                           | Add exact HA origin to Chrome secure-origin override       |
| Provisioning ADB library will not load               | Browser cannot reach external module host       | Temporarily allow provisioning workstation Internet access |
| Root verification initially fails                    | Magisk daemon still initializing                | Allow wizard retries                                       |
| Echo has saved Wi-Fi but no route                    | Not actually associated                         | Check `wpa_cli`                                            |
| `ip route get` reports unreachable                   | No valid network path                           | Restore Wi-Fi before troubleshooting EchoMuse              |
| `wpa_cli` does not show useful state                 | Wrong control socket                            | Use `/data/misc/wifi/sockets`                              |
| Network exists but remains scanning                  | Saved network not selected                      | `select_network 0`                                         |
| Wi-Fi breaks after reboot                            | Amazon Wi-Fi manager interferes                 | Stop `smarthomewifid`                                      |
| mDNS query is visible but EchoMuse is not discovered | Reflection/publication issue                    | Verify Avahi and DNS-SD                                    |
| `avahi-publish-address` is unreliable                | Wrong publication mechanism for this use        | Use `avahi-publish-service`                                |
| Wake word works but voice turn stops                 | ESPHome endpoint not connected to HA            | Add EchoMuse ESPHome device                                |
| Whisper transcription is correct                     | Audio/STT path is already healthy               | Debug later Assist stages instead                          |

---

## Recovery Workflow

### 1. Verify ADB

```bash
sudo adb devices -l
```

### 2. Verify Wi-Fi

```bash
sudo adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 status'
```

Expected:

```text
wpa_state=COMPLETED
```

### 3. Stop smarthomewifid if Required

```bash
sudo adb shell su -c \
'getprop init.svc.smarthomewifid'
```

If running:

```bash
sudo adb shell su -c \
'stop smarthomewifid'
```

### 4. Select IoT Network

```bash
sudo adb shell su -c \
'wpa_cli -p /data/misc/wifi/sockets -i wlan0 select_network 0'
```

### 5. Verify IP

Expected:

```text
10.20.90.60
```

### 6. Verify Route

```bash
sudo adb shell su -c \
'ip route get 10.20.30.10'
```

Expected:

```text
10.20.30.10 via 10.20.90.1 dev wlan0 src 10.20.90.60
```

Do not continue until this works.

### 7. Verify EchoMuse

```bash
sudo adb shell su -c \
'ps | grep /data/local/bin/server'
```

### 8. Check EchoMuse Log

```bash
sudo adb shell su -c \
'cat /tmp/server.log' | tail -60
```

### 9. Verify Avahi

```bash
systemctl status avahi-daemon --no-pager
```

### 10. Verify mDNS Publisher

```bash
systemctl status echomuse-mdns.service --no-pager
```

### 11. Verify DNS-SD

```bash
avahi-browse -rt _emcontroller._tcp
```

### 12. Verify ESPHome in Home Assistant

Confirm the EchoMuse voice satellite is still connected.

### 13. Test Wake Word

Say:

```text
Hey Jarvis
```

Confirm a wake-word detection event occurs.

### 14. Test STT

Speak a simple request.

Confirm faster-whisper produces the correct transcript.

### 15. Troubleshoot the Correct Layer

If STT succeeds, do not restart with:

```text
Wi-Fi
mDNS
EchoMuse discovery
microphone
OpenWakeWord
```

unless one of those tests now shows a new failure.

Continue at the Home Assistant Assist conversation stage.

---

## Replacement Device Build Order

For another RS03QR:

```text
1. Confirm RS03QR / biscuit
2. Prepare Linux
3. Install Amonet dependencies
4. Disable ModemManager
5. Enter supported fastboot path if available
6. If unsupported, use documented BootROM short
7. Run bootrom-step.sh
8. Release short only when instructed
9. Run fastboot-step.sh
10. Verify TWRP
11. Wipe data and cache
12. Install known-good Fire OS 5
13. Install f1r30s
14. Boot Fire OS
15. Verify ADB
16. Deploy EchoMuse Controller
17. Open EchoMuse provisioning in Chrome
18. Resolve WebUSB secure-context requirement if needed
19. Allow browser dependency access if required
20. Run adb kill-server before WebUSB
21. Connect Dot through EchoMuse wizard
22. Select Magisk-v17.3.zip
23. Complete all provisioning stages
24. Verify IoT Wi-Fi
25. Verify route to EchoMuse
26. Verify mDNS
27. Approve device in EchoMuse
28. Add ESPHome satellite in Home Assistant
29. Test wake word
30. Test faster-whisper
31. Power-cycle device
32. Verify Wi-Fi persistence
33. Verify EchoMuse reconnection
34. Perform final voice test
```

---

## Credits and Upstream Projects

This project and documentation build upon the work of the developers and community members who made repurposing the Amazon Echo Dot 2nd Generation possible.

### Echo Dot 2 Unlock, Root, TWRP, and Recovery

The bootloader unlock, rooting, TWRP installation, BootROM recovery, and related low-level procedures are based on the work documented by the XDA Developers community:

XDA Developers - Unlock/Root/TWRP/Unbrick Amazon Echo Dot 2nd Gen (2016) "biscuit"

https://xdaforums.com/t/unlock-root-twrp-unbrick-amazon-echo-dot-2nd-gen-2016-biscuit.4761416/

Credit belongs to the original developers, researchers, and contributors responsible for the Amonet-based unlock and recovery work documented there.

### EchoMuse

The conversion of the unlocked Echo Dot into a Home Assistant-compatible local voice satellite is made possible by EchoMuse, developed and maintained by wilbowes and project contributors.

EchoMuse Documentation:

https://github.com/wilbowes/EchoMuse/blob/main/docs/README.md

EchoMuse Repository:

https://github.com/wilbowes/EchoMuse

EchoMuse provides the device-side software, Home Assistant controller, provisioning workflow, ESPHome voice-satellite integration, wake-word functionality, and supporting components used in this deployment.

### Attribution

This documentation describes the installation, integration, network configuration, troubleshooting, and operational experience of deploying these technologies together. It does not claim authorship or ownership of Amonet, EchoMuse, TWRP, Magisk, or other third-party projects referenced herein.

Refer to each upstream project for its respective authorship, current documentation, and source code.

---

## Related Search Keywords

amazon-echo-dot-2, rs03qr, echomuse, home-assistant, esphome, voice-assistant, amonet, bootrom, twrp, magisk, fireos-5, openwakeword, faster-whisper, piper, avahi, mdns, iot-security, network-segmentation

---

## Revision Control

| Version   | Date       | Summary                                                                                                                                                                                                                                                                                                                            | Author      |
| --------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| **1.0.0** | 2026-08-22 | Public-safe operational guide for converting an Amazon Echo Dot 2 RS03QR into an EchoMuse-backed Home Assistant voice satellite. Internal IP addresses, device identifiers, wireless identifiers, authentication material, and other deployment-specific information have been removed or replaced with consistent example values. | projectfong |
