# Smart Home Documentation

The `smarthome/` directory contains implementation, integration, troubleshooting, migration, and validation guides for Home Assistant OS, Matter, Thread, ESPHome, local device integrations, voice infrastructure, and segmented IoT networks.

For condensed deployment procedures, see the [Quickstart Guides](quickstart/).

### Home Assistant and Network Architecture

* [Home Assistant OS on Proxmox](haos-proxmox-deploy.md) - Deploys Home Assistant OS as a virtual machine on Proxmox VE and documents the supporting virtualization architecture.
* [Home Assistant OS Proxmox VM to Bare Metal Migration](haos-pvevm-to-baremetal.md) - Documents migration of Home Assistant OS from a Proxmox VE virtual machine to dedicated bare-metal hardware, including backup restoration, networking, and post-migration validation.
* [Home Assistant OS External SSH Access](haos-ssh-external-access.md) - Documents configuration and validation of direct external SSH access to Home Assistant OS.
* [Matter over Segmented Wi-Fi](ha-matter-segmented-network.md) - Documents Matter operation across segmented Home Assistant and IoT networks, including routing, firewall policy, mDNS, and validation.
* [Aqara H2 Matter-over-Thread](aqara-h2-haos-mot.md) - Documents installation and local Home Assistant integration of the Aqara Light Switch H2 US using Matter-over-Thread in a segmented network.

### ESPHome and Local Device Integrations

* [ratgdo Chamberlain ESPHome](ratgdo-chamberlain-esphome.md) - Documents local Chamberlain garage-door integration using ratgdo, ESPHome, and Home Assistant.
* [Tapo H100 Doorbell Integration](tapo-h100-haos-doorbell.md) - Documents local Home Assistant integration and automation of a Tapo H100-based doorbell environment.
* [Google Nest Device Access](google-nest-device-access-with-haos.md) - Documents Google Nest Device Access integration with Home Assistant OS while preserving a locally managed Home Assistant architecture.

### Voice Infrastructure and EchoMuse

* [Echo Dot 2 Voice Satellite](echo-dot2-haos.md) - Documents repurposing an Amazon Echo Dot 2 as an EchoMuse-based local Home Assistant voice satellite.
* [Echo Dot 2 Wi-Fi Disconnect Root Cause Analysis](echo-dot2-intermittent-troubleshooting.md) - Documents investigation of recurring EchoMuse disconnects and identifies the Fire OS `WifiDiagsUtil` network-bounce mechanism as the primary recurring Wi-Fi interruption source.

## Revision Control

| Version   | Date       | Summary                                                                                                 | Author      |
| --------- | ---------- | ------------------------------------------------------------------------------------------------------- | ----------- |
| **1.0.0** | 2026-09-02 | Updated documentation index with quickstart, HAOS bare-metal migration, and external SSH documentation. | projectfong |
