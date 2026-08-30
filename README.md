# projectfong/docs - Public Technical Documentation

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

`projectfong/docs` is a public-safe technical documentation repository covering systems architecture, virtualization, networking, endpoint management, smart home infrastructure, and operational maintenance.

The repository documents implementation decisions, configuration procedures, troubleshooting, validation, and lessons learned while removing private addressing, credentials, internal identifiers, and other environment-specific information.

Documentation emphasizes secure design, segmentation, validation, reproducibility, and operational understanding.

---

## Purpose

The goal of this repository is to turn completed technical work into documentation that can be safely referenced, reproduced, and shared.

Documentation is intended to:

* Record complete implementation procedures rather than only final configurations.
* Explain why architectural and configuration decisions were made.
* Preserve troubleshooting steps and failure conditions that contributed to the final solution.
* Provide validation procedures so configurations can be independently verified.
* Maintain sanitized examples that are safe for public repositories.
* Create repeatable procedures that can be followed by someone unfamiliar with the original implementation.
* Document interactions between infrastructure, networking, security, endpoints, virtualization, and smart home systems.
* Provide searchable technical references for future troubleshooting and implementation work.

This repository contains documentation and reference material. Individual projects, scripts, configurations, or software may be maintained in separate repositories.

---

## Planned Documentation Structure

The repository is organized by technical domain as documentation is added.

```text
docs/
├── architecture/
│   ├── network/
│   ├── security/
│   └── diagrams/
├── virtualization/
├── networking/
├── endpoints/
├── smarthome/
├── maintenance/
├── assets/
│   ├── diagrams/
│   └── screenshots/
└── README.md
```

Not every planned directory may be populated yet.

Directories are created as documentation for each technical area is completed and prepared for public release.

### `architecture/`

System-level architecture and design documentation.

Topics may include:

* Global network topology
* Logical architecture
* Security boundaries
* Trust relationships
* Network segmentation models
* Threat models
* RFC 5737 documentation addressing
* Architecture diagrams
* Design decisions

### `virtualization/`

Virtualization platforms, hypervisors, compute infrastructure, and associated storage.

Topics may include:

* Proxmox VE
* KVM
* Virtual machines
* Containers
* Clustering
* Storage pools
* ZFS
* Virtual networking
* Backup and recovery

### `networking/`

Network infrastructure configuration, segmentation, routing, and security controls.

Topics may include:

* FortiGate
* VLAN design
* Inter-VLAN routing
* Firewall policies
* Static routing
* DNS
* DHCP
* mDNS
* Network isolation
* IoT segmentation
* Cross-VLAN service discovery
* Troubleshooting procedures

### `endpoints/`

Endpoint deployment, configuration, management, and validation.

Topics may include:

* Windows 11
* Offline Windows OOBE
* Linux workstations
* Mobile devices
* Local account provisioning
* Storage encryption validation
* Network configuration
* Endpoint hardening

### `smarthome/`

Local smart home infrastructure, automation systems, embedded devices, and voice interfaces.

Topics may include:

* Home Assistant OS
* Home Assistant
* Matter
* Thread
* OpenThread Border Router
* ESPHome
* ESP32
* IoT network segmentation
* Cross-VLAN communication
* mDNS and service discovery
* Voice satellites
* Repurposed hardware
* Local voice processing
* Smart switches
* Smart lighting
* Automation infrastructure

### `maintenance/`

Operational procedures required to maintain, recover, validate, and safely document systems.

Topics may include:

* Backup strategies
* Disaster recovery
* Recovery runbooks
* Configuration validation
* Upgrade procedures
* Documentation sanitization
* Public release review
* Troubleshooting workflows

### `assets/`

Sanitized supporting material referenced by documentation.

```text
assets/
├── diagrams/
└── screenshots/
```

Shared architecture diagrams belong under `assets/diagrams/`.

Sanitized screenshots used by documentation belong under `assets/screenshots/`.

Documentation-specific assets may instead be stored near the document that uses them when doing so improves organization.

---

## Documentation Approach

Documentation in this repository is intended to record more than a successful end state.

Where applicable, a technical guide should document:

1. What was being implemented.
2. Why the implementation was needed.
3. Hardware and software requirements.
4. Relevant architecture.
5. Dependencies.
6. Initial configuration.
7. Commands or API calls used.
8. Configuration changes.
9. Problems encountered.
10. Cause of each identified problem.
11. Troubleshooting performed.
12. Final fix.
13. Validation procedures.
14. Expected results.
15. Security considerations.
16. Public sanitization performed.

This provides enough context to reproduce an implementation without relying on undocumented knowledge from the original environment.

The objective is not simply to record commands.

The objective is to preserve enough engineering context to explain:

```text
What was built?
Why was it built this way?
What failed?
Why did it fail?
How was the failure identified?
What corrected it?
How was the final implementation validated?
```

---

## Public-Safe Documentation

Material published here is intentionally sanitized before publication.

Public documentation must not expose private operational information simply because that information was useful during the original implementation.

### Information That Must Be Sanitized

Examples include:

* Public IP addresses associated with private infrastructure
* Actual private network addressing
* Internal IPv4 and IPv6 prefixes
* Internal host addresses
* Internal hostnames
* Internal DNS domains
* Device serial numbers
* USB serial identifiers
* MAC addresses
* Hardware-derived identifiers
* Thread network credentials
* Thread network identifiers when unnecessarily environment-specific
* Matter setup credentials
* Usernames
* Email addresses
* Authentication tokens
* API keys
* Passwords
* Private certificates
* Private keys
* VPN configuration secrets
* Wireless credentials
* Organization-specific identifiers
* Internal firewall object names when they disclose architecture
* Internal firewall interface names when unnecessary
* Screenshots containing sensitive environment information

Sanitization should preserve the technical meaning of an example.

For example, documentation requiring IPv4 addresses should use appropriate documentation networks rather than arbitrary addresses.

RFC 5737 reserves the following IPv4 networks for documentation:

```text
192.0.2.0/24
198.51.100.0/24
203.0.113.0/24
```

These ranges allow network examples to remain technically meaningful without publishing operational addresses.

For values that cannot be represented cleanly using documentation addressing, descriptive placeholders may be used.

Examples:

```text
<HA_IPV6_PREFIX>
<IOT_IPV6_PREFIX>
<THREAD_OMR_PREFIX>
<SERVER_IPV6>
<FIREWALL_INTERFACE>
<DEVICE_ID>
```

### Sanitization Principle

A sanitized document should still answer:

```text
What connects to what?
Why is the connection required?
Which protocol is used?
Which port or service behavior is required?
Which direction does traffic flow?
Which security boundary is crossed?
How can the configuration be validated?
```

Removing sensitive information should not remove the engineering logic needed to understand the implementation.

---

## Security and Isolation

Security documentation focuses on architecture and technical controls without publishing unnecessary information about private infrastructure.

Depending on the system being documented, this may include:

* Network segmentation
* Least-privilege access
* Explicit firewall policy
* Local-first services
* Controlled external connectivity
* Authentication boundaries
* Service isolation
* Encryption
* Logging
* Configuration validation
* Backup and recovery
* Failure behavior
* Reduced external trust dependencies

Documentation may reference established security frameworks or industry guidance when those concepts help explain an architectural decision.

Such references describe engineering principles and design considerations. They do not represent certification, formal assessment, or a compliance claim for systems documented in this public repository.

---

## Commands and Configuration Examples

Commands included in documentation should be complete enough to reproduce the documented operation.

A command should not be included without sufficient context to understand:

* Where it is executed
* Which account or privilege level is required
* What it changes
* Why it is required
* What successful execution looks like
* How the result can be verified

Example structure:

```bash
command --required-option value
```

The surrounding documentation should explain the command rather than expecting the reader to infer its purpose.

Destructive or environment-specific commands require additional explanation before execution.

Configuration examples should use sanitized values while remaining syntactically valid whenever possible.

---

## Validation

Configuration is not considered complete merely because a command executed successfully.

Documentation should include validation appropriate to the technology being configured.

Validation may include:

```text
Service status
Network reachability
Route verification
DNS resolution
Port connectivity
Firewall logging
Application logs
Authentication testing
Configuration inspection
Restart persistence
Reboot persistence
Failure testing
Recovery testing
```

Where practical, expected results should accompany validation commands.

A successful validation should demonstrate that the intended system behavior works rather than only confirming that configuration syntax was accepted.

---

## Troubleshooting Documentation

Troubleshooting is retained when it materially contributed to the working implementation.

Useful troubleshooting documentation identifies:

```text
Symptom
Cause
Investigation
Fix
Validation
```

Failed approaches may be included when they explain an important limitation or prevent the same troubleshooting path from being repeated.

Unrelated experimentation is excluded from the final public guide.

The objective is not to preserve every action taken during experimentation.

The objective is to preserve the information required to understand how the working implementation was reached.

### Follow the Evidence

When troubleshooting networking or distributed systems, validation should occur at each relevant layer rather than assuming that one successful component proves the complete path.

Examples may include:

```text
Physical / radio connectivity
        |
        v
Link or interface state
        |
        v
IP addressing
        |
        v
Routing
        |
        v
Firewall policy
        |
        v
Service discovery
        |
        v
Application protocol
        |
        v
Application behavior
```

Packet captures, logs, routing tables, firewall debug output, and service state should be preferred over assumptions when identifying a failure.

---

## Visual Documentation

Architecture diagrams and screenshots may be included when they improve understanding.

Visual material must be sanitized using the same requirements as written documentation.

When an image is included, the surrounding document should also describe the important information shown in the image.

Example:

```markdown
![Network segmentation diagram](../assets/diagrams/network-segmentation.png)

The diagram shows the management, trusted client, server, and IoT network segments separated by firewall boundaries. Inter-VLAN communication is denied by default and explicitly permitted only for documented service flows.
```

This keeps important architectural information searchable and understandable when the image is unavailable.

---

## Documentation Workflow

A typical publication workflow is:

```text
Private implementation
        |
        v
Technical notes
        |
        v
Working configuration
        |
        v
Validation
        |
        v
Documentation
        |
        v
Sanitization
        |
        v
Public-safe review
        |
        v
Git commit
        |
        v
Public repository
```

Before publication:

1. Confirm that the documented implementation actually worked.
2. Remove unrelated experimentation.
3. Preserve troubleshooting that materially contributed to the solution.
4. Replace environment-specific values with safe examples or descriptive placeholders.
5. Review commands for destructive or environment-specific behavior.
6. Verify links and references.
7. Review screenshots and diagrams.
8. Confirm that no credentials or secrets are present.
9. Review addressing, identifiers, hostnames, serial values, and other environment-specific information.
10. Confirm that the sanitized procedure remains technically reproducible.
11. Commit the reviewed documentation.

---

## Repository Scope

This repository is intended for technical documentation rather than acting as a monolithic source repository.

A topic may move into a dedicated repository when it develops substantial independent:

* Source code
* Automation
* Infrastructure-as-code
* Container definitions
* Build systems
* Release artifacts
* Hardware designs
* Firmware
* Testing infrastructure

The `docs` repository can continue to contain architectural or operational documentation that references those projects.

This keeps the repository useful as a centralized technical knowledge base without forcing unrelated source code into the same project.

---

## Search Keywords

Related technical terms used throughout this repository may include:

**Systems Architecture**, **Homelab**, **Infrastructure**, **Network Architecture**, **Network Segmentation**, **VLAN**, **Firewall**, **FortiGate**, **Routing**, **IPv4**, **IPv6**, **DNS**, **DHCP**, **mDNS**, **Proxmox VE**, **KVM**, **ZFS**, **Windows 11**, **Linux**, **Home Assistant**, **HAOS**, **Matter**, **Matter-over-Wi-Fi**, **Matter-over-Thread**, **Thread**, **OpenThread**, **OTBR**, **ESPHome**, **ESP32**, **IoT**, **Smart Home**, **Voice Satellite**, **Disaster Recovery**, **Backup**, **Runbook**, **Cybersecurity**, **Zero Trust**, **Technical Documentation**, **Troubleshooting**, **Validation**, **Sanitization**

---

## Revision Control

| Version | Date | Summary | Author |
| --- | --- | --- | --- |
| **1.0.0** | 2026-08-23 | Initial publication of `README.md` and baseline documentation structure for `projectfong/docs`. | projectfong |
| **1.1.0** | 2026-08-30 | Refined repository scope and documentation philosophy, clarified the planned directory structure, expanded public-safe sanitization guidance, added IPv6 and identifier redaction practices, and simplified security-framework language. | projectfong |
