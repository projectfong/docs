# projectfong/docs - Public Technical Documentation

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

`projectfong/docs` is a public-safe technical documentation repository covering systems architecture, virtualization, networking, endpoint management, smart home infrastructure, and operational maintenance.

The repository documents implementation decisions, configuration procedures, troubleshooting, validation, and lessons learned while removing private addressing, credentials, internal identifiers, and other environment-specific information. Documentation follows established cybersecurity frameworks and best practices, including concepts found in NIST SP 800-171, with an emphasis on secure design, segmentation, validation, reproducibility, and operational understanding.

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

Next Step: Browse the documentation tree below to locate the relevant technical area.

---

## Documentation Structure

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
* ESPHome
* ESP32
* IoT network segmentation
* Cross-VLAN communication
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

Validation Result: The repository separates documentation by operational domain while maintaining shared locations for reusable visual material.

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

Next Step: Review the applicable document's requirements and architecture sections before beginning an implementation.

---

## Public-Safe Documentation

Material published here is intentionally sanitized before publication.

Public documentation must not expose private operational information simply because that information was useful during the original implementation.

### Information That Must Be Sanitized

Examples include:

* Public IP addresses associated with private infrastructure
* Actual private network addressing
* Internal hostnames
* Internal DNS domains
* Device serial numbers
* MAC addresses
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

### Sanitization Principle

A sanitized document should still answer:

```text
What connects to what?
Why is the connection required?
Which protocol is used?
Which port is required?
Which direction does traffic flow?
Which security boundary is crossed?
How can the configuration be validated?
```

Removing sensitive information should not remove the engineering logic needed to understand the implementation.

Validation Result: Public examples retain their architectural meaning without exposing operational environment details.

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

Documentation may reference established cybersecurity frameworks and best practices, including relevant NIST SP 800-171 concepts, when those concepts help explain an architectural decision.

Such references describe engineering principles and design considerations. They do not represent certification or a compliance claim for systems documented in this public repository.

Next Step: Validate security-sensitive examples during sanitization before committing them to the public repository.

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

Validation Result: Commands and configuration examples are documented with enough context to understand both execution and verification.

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

Next Step: Complete all documented validation procedures before treating an implementation as operational.

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

The objective is not to preserve every action taken during experimentation. The objective is to preserve the information required to understand how the working implementation was reached.

Validation Result: Troubleshooting history documents technically relevant failures without overwhelming the final implementation procedure.

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

Next Step: Review every screenshot and diagram for identifying information before publication.

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
4. Replace environment-specific values with safe examples.
5. Review commands for destructive or environment-specific behavior.
6. Verify links and references.
7. Review screenshots and diagrams.
8. Confirm that no credentials or secrets are present.
9. Confirm that the sanitized procedure remains technically reproducible.
10. Commit the reviewed documentation.

Validation Result: Documentation reaches the public repository only after technical validation and sanitization.

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

Next Step: Create dedicated repositories when a documented system becomes an independently maintained software or hardware project.

---

## Search Keywords

Related technical terms used throughout this repository may include:

**Systems Architecture**, **Homelab**, **Infrastructure**, **Network Architecture**, **Network Segmentation**, **VLAN**, **Firewall**, **FortiGate**, **Routing**, **DNS**, **DHCP**, **mDNS**, **Proxmox VE**, **KVM**, **ZFS**, **Windows 11**, **Linux**, **Home Assistant**, **HAOS**, **Matter**, **ESPHome**, **ESP32**, **IoT**, **Smart Home**, **Voice Satellite**, **Disaster Recovery**, **Backup**, **Runbook**, **Cybersecurity**, **NIST SP 800-171**, **Zero Trust**, **Technical Documentation**, **Troubleshooting**, **Validation**, **Sanitization**

---

## Distribution and Copyright

Copyright (c) 2026 Fong.

Permission is granted to redistribute this documentation in its original, unmodified form, provided appropriate credit is given to projectfong and a link to the projectfong GitHub page is included:

https://github.com/projectfong/

---

## Revision Control

| Version   | Date       | Summary                                                                                         | Author      |
| --------- | ---------- | ----------------------------------------------------------------------------------------------- | ----------- |
| **1.0.0** | 2026-08-23 | Initial publication of `README.md` and baseline documentation structure for `projectfong/docs`. | projectfong |
