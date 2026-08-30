## Smart Home Documentation

The `smarthome/` directory contains implementation, integration, troubleshooting, and validation guides for Home Assistant OS, Matter, Thread, ESPHome, local device integrations, voice infrastructure, and segmented IoT networks.

### Home Assistant and Network Architecture

* [Home Assistant OS on Proxmox](https://github.com/projectfong/docs/blob/main/smarthome/haos-proxmox-deploy.md) - Deploys Home Assistant OS as a virtual machine on Proxmox VE and documents the supporting virtualization architecture.
* [Matter over Segmented Wi-Fi](https://github.com/projectfong/docs/blob/main/smarthome/ha-matter-segmented-network.md) - Documents Matter operation across segmented Home Assistant and IoT networks, including routing, firewall policy, mDNS, and validation.
* [Aqara H2 Matter-over-Thread](https://github.com/projectfong/docs/blob/main/smarthome/aqara-h2-haos-mot.md) - Documents installation and local Home Assistant integration of the Aqara Light Switch H2 US using Matter-over-Thread in a segmented network.

### ESPHome and Local Device Integrations

* [ratgdo Chamberlain ESPHome](https://github.com/projectfong/docs/blob/main/smarthome/ratgdo-chamberlain-esphome.md) - Documents local Chamberlain garage-door integration using ratgdo, ESPHome, and Home Assistant.
* [Tapo H100 Doorbell Integration](https://github.com/projectfong/docs/blob/main/smarthome/tapo-h100-haos-doorbell.md) - Documents local Home Assistant integration and automation of a Tapo H100-based doorbell environment.
* [Google Nest Device Access](https://github.com/projectfong/docs/blob/main/smarthome/google-nest-device-access-with-haos.md) - Documents Google Nest Device Access integration with Home Assistant OS while preserving a locally managed Home Assistant architecture.

### Voice Infrastructure and EchoMuse

* [Echo Dot 2 Voice Satellite](https://github.com/projectfong/docs/blob/main/smarthome/echomuse-echodot2-haos.md) - Documents repurposing an Amazon Echo Dot 2 as an EchoMuse-based local Home Assistant voice satellite.
* [Echo Dot 2 Wi-Fi Disconnect Root Cause Analysis](https://github.com/projectfong/docs/blob/main/smarthome/echomuse-wifi-disconnect-investigation.md) - Documents investigation of recurring EchoMuse disconnects and identifies the Fire OS `WifiDiagsUtil` network-bounce mechanism as the primary recurring Wi-Fi interruption source.

The complete Smart Home documentation directory is available at:

https://github.com/projectfong/docs/tree/main/smarthome

Validation Result: The Smart Home documentation index provides direct links to the completed Home Assistant, Matter, Thread, ESPHome, device-integration, and EchoMuse guides maintained under `smarthome/`.
