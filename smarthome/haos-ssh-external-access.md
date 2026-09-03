````markdown
# Home Assistant OS Terminal & SSH - Enabling External SSH Access

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

This document records the troubleshooting and configuration required to enable direct external SSH access to the Home Assistant OS `Terminal & SSH` application when the browser-based Web Terminal is already functional but a normal SSH client receives `Connection refused`.

The important distinction is that the Web Terminal and external SSH access are separate access paths. A working Web Terminal does not confirm that TCP/22 is exposed by the application. In this deployment, external SSH became available only after the SSH port was explicitly enabled in the application's Network section and that section was saved independently.

## Purpose

The purpose of this procedure is to document the specific condition where:

- The Home Assistant OS `Terminal & SSH` application is installed.
- The browser-based Web Terminal works.
- Commands can already be executed through the Home Assistant interface.
- HAOS responds normally on the network.
- A remote workstation cannot establish an SSH connection.
- TCP/22 initially reports as unavailable.
- The failure can appear similar to a firewall, VLAN, routing, or service problem.
- The actual issue is that the application's external SSH port has not been enabled.
- The Network configuration must be saved separately from the application's other configuration sections.

This procedure also provides a repeatable validation method for distinguishing application configuration problems from network connectivity problems.

**Validation Result:** The issue was isolated to SSH port exposure in the `Terminal & SSH` application rather than basic network reachability.

---

## Environment

The following values were observed during troubleshooting.

| Component             | Configuration             | Notes                                                          |
| --------------------- | ------------------------- | -------------------------------------------------------------- |
| Platform              | Home Assistant OS         | Existing HAOS deployment                                       |
| HAOS address          | `<HAOS_IP>`               | Address used during testing                                    |
| Application           | `Terminal & SSH`          | Home Assistant application providing terminal and SSH access   |
| Web Terminal          | Working                   | Accessible through Home Assistant ingress                      |
| External SSH          | Initially unavailable     | SSH client could not connect                                   |
| SSH port              | `22/tcp`                  | External port configured during troubleshooting                |
| Test workstation      | Windows                   | PowerShell used for network validation                         |
| ICMP connectivity     | Working                   | HAOS responded with approximately 1 ms latency                 |
| TCP/22 connectivity   | Initially failed          | `TcpTestSucceeded : False`                                     |
| Authentication        | Password configured       | Used in the isolated administrative environment                |
| Network configuration | Separate application section | Required its own Save operation                             |

**Validation Result:** HAOS was reachable from the administrative workstation before external SSH was enabled.

---

## Architecture

The relevant access paths are:

```text
Administrative Workstation
|
+-- Web Browser
|   |
|   +-- Home Assistant
|       |
|       +-- Terminal & SSH
|           |
|           +-- Web Terminal
|               |
|               +-- Working before TCP/22 was enabled
|
+-- SSH Client
    |
    +-- TCP/22
        |
        +-- HAOS
            |
            +-- Terminal & SSH
                |
                +-- External SSH service
```

The browser-based terminal and direct SSH access use different connection paths.

The Web Terminal operates through the Home Assistant interface and can therefore remain functional even when the application does not expose an external SSH port.

This distinction is important during troubleshooting because a functional Web Terminal only confirms that the application and Home Assistant ingress path are operational.

It does not confirm:

```text
HAOS-IP:22
```

is listening for external SSH connections.

**Validation Result:** Web Terminal availability and external SSH availability are separate conditions.

---

# Initial Condition

## 1. Confirm the Web Terminal Works

The `Terminal & SSH` application was already installed and operational through Home Assistant.

The browser-based terminal could be opened and commands could be executed.

This established that:

- The application was installed.
- The application was running.
- Home Assistant ingress could reach the application.
- Interactive shell access was available through the browser.

The browser terminal was usable, but copying and pasting larger commands or log output was less practical than using a normal SSH client.

The goal was therefore to enable direct SSH access from the administrative workstation.

**Validation Result:** The application was operational before any external SSH changes were made.

---

# External Connectivity Test

## 2. Attempt an SSH Connection

From the administrative workstation, an SSH connection was attempted against the HAOS address.

Example:

```text
ssh root@<HAOS_IP>
```

The connection failed because TCP/22 was not accepting connections.

The exact authentication username can depend on the configuration of the `Terminal & SSH` application. Use the username configured in the application.

The important result at this stage was not authentication failure.

The SSH TCP connection itself could not be established.

**Validation Result:** External SSH failed before reaching the authentication stage.

---

## 3. Test TCP/22 with PowerShell

PowerShell was used to separate basic network reachability from TCP service availability.

Run:

```text
Test-NetConnection <HAOS_IP> -Port 22
```

The observed output was:

```text
WARNING: TCP connect to (<HAOS_IP> : 22) failed

ComputerName           : <HAOS_IP>
RemoteAddress          : <HAOS_IP>
RemotePort             : 22
InterfaceAlias         : <WORKSTATION_INTERFACE>
SourceAddress          : redacted
PingSucceeded          : True
PingReplyDetails (RTT) : 1 ms
TcpTestSucceeded       : False
```

The two key fields were:

```text
PingSucceeded    : True
TcpTestSucceeded : False
```

### Interpretation

The successful ICMP response established that the workstation could reach HAOS.

The failed TCP result established that port 22 was not accepting the connection.

The observed condition was:

```text
Administrative Workstation
        |
        +-- ICMP
        |     |
        |     +-- SUCCESS
        |
        +-- TCP/22
              |
              +-- FAILED
```

This reduced the troubleshooting scope substantially.

The surrounding network path was already functional enough to reach the HAOS host.

**Validation Result:** HAOS was reachable, but TCP/22 was not exposed or listening externally.

---

# Root Cause

## 4. Web Terminal Access Does Not Enable External SSH Automatically

The working browser terminal initially made the application appear fully configured.

It was not.

The `Terminal & SSH` application exposes additional network services through its own Network configuration section.

The relevant structure is:

```text
Terminal & SSH
|
+-- Application configuration
|
+-- Web Terminal through Home Assistant ingress
|
+-- External Network configuration
    |
    +-- SSH Port
        |
        +-- TCP/22
```

The Web Terminal could function while the SSH port remained disabled.

This produced the exact observed behavior:

```text
Browser
|
+-- Home Assistant
    |
    +-- Terminal & SSH
        |
        +-- WORKING


SSH Client
|
+-- <HAOS_IP>:22
    |
    +-- CONNECTION FAILED
```

### Cause

The application's external SSH port had not been assigned in the Network section.

The visible:

```text
22/tcp
```

label identifies the application's SSH container port.

It does not by itself confirm that a host port has been assigned.

The host-side SSH Port field must contain a port number.

**Validation Result:** The missing host-port assignment prevented direct SSH access.

---

# Configuration Procedure

## 5. Open the Terminal & SSH Application

In Home Assistant, open:

```text
Settings
-> Apps
-> Terminal & SSH
```

Open the application configuration page.

The application contains multiple configuration areas, including:

```text
Options

Network
```

The external SSH port is controlled from:

```text
Network
```

**Next Step:** Display the disabled network ports.

---

## 6. Enable Show Disabled Ports

In the Network section, enable:

```text
Show disabled ports
```

When disabled ports are hidden, the SSH port configuration may not be visible.

After enabling the option, the SSH Port entry becomes available.

The interface displays:

```text
SSH Port
```

with:

```text
22/tcp
```

shown as the underlying application port.

The editable field to the left is the host-side port assignment.

If that field is blank, external SSH is not exposed.

The effective configuration before the fix was:

```text
Host Port:      <blank>
Container Port: 22/tcp
```

**Validation Result:** The SSH container port existed, but no external host port was assigned.

---

## 7. Configure the SSH Host Port

Enter:

```text
22
```

in the SSH Port field.

The resulting mapping becomes:

```text
HAOS host
<HAOS_IP>
|
+-- TCP/22
    |
    +-- Terminal & SSH
        |
        +-- SSH service
```

The Network section should now visually show:

```text
22                 22/tcp
SSH Port
```

where:

```text
22
```

on the left is the host-side port assignment, and:

```text
22/tcp
```

on the right is the application/container port.

**Next Step:** Save the Network section.

---

# Important UI Behavior

## 8. Save Each Configuration Section Independently

A critical behavior discovered during configuration is that the Home Assistant application interface provides separate Save controls for individual configuration sections.

The Options section and Network section must be treated independently.

The application page effectively behaves as:

```text
Terminal & SSH Configuration
|
+-- Options
|   |
|   +-- Authentication settings
|   |
|   +-- Save
|
+-- Network
    |
    +-- SSH Port
    |
    +-- Save
```

Saving the Options section does not automatically save changes made in the Network section.

During troubleshooting, the authentication settings had been saved, but the port assignment had not yet been committed because the Network section required its own Save action.

This resulted in:

```text
Password configured
        |
        +-- Saved
        |
        +-- SSH authentication configuration valid


SSH Port configured
        |
        +-- Not yet saved
        |
        +-- TCP/22 remained unavailable
```

After the Network section was saved separately, the host port became active.

### Operational Lesson

When changing Home Assistant application settings, verify whether each configuration card or section has its own Save control.

Do not assume that a Save action elsewhere on the page commits every pending change.

**Validation Result:** Saving the Network section independently was required before TCP/22 became available.

---

## 9. Restart Terminal & SSH

After changing network exposure settings, restart the `Terminal & SSH` application.

Navigate to the application information page and restart it.

The restart ensures that the current configuration is applied cleanly.

The intended state is:

```text
Terminal & SSH
|
+-- Authentication configured
|
+-- Host SSH port = 22
|
+-- Application restarted
|
+-- SSH service reachable through HAOS TCP/22
```

**Validation Result:** The application was restarted with the saved SSH port configuration.

---

# Verification

## 10. Retest TCP/22

Return to the Windows administrative workstation.

Run:

```text
Test-NetConnection <HAOS_IP> -Port 22
```

Before the configuration fix, the result was:

```text
PingSucceeded    : True
TcpTestSucceeded : False
```

After successful port exposure, the expected TCP result is:

```text
TcpTestSucceeded : True
```

This validates the path:

```text
Windows Workstation
|
+-- TCP/22
    |
    +-- <HAOS_IP>
        |
        +-- Terminal & SSH
```

If TCP succeeds, the listener and network path are working.

Authentication can then be tested separately.

**Validation Result:** TCP/22 became reachable after the SSH host port was saved and applied.

---

## 11. Connect Using SSH

Connect using the configured SSH username.

General syntax:

```text
ssh <configured-user>@<HAOS_IP>
```

If the application is configured to use `root` as the SSH username, the command is:

```text
ssh root@<HAOS_IP>
```

When password authentication is enabled, the SSH client prompts for the configured password.

A successful connection confirms:

```text
Workstation
|
+-- TCP/22
    |
    +-- HAOS
        |
        +-- Terminal & SSH
            |
            +-- Authentication
                |
                +-- Interactive shell
```

This provides a standard SSH session and avoids relying on the browser-based Web Terminal for routine command entry, log collection, and copy/paste operations.

**Validation Result:** External SSH access was successfully established.

---

# Authentication Considerations

## Password Authentication

Password authentication can be appropriate in a controlled internal administrative network when:

- The service is not exposed to the public Internet.
- Access is restricted by network segmentation.
- The password is strong and unique.
- Administrative systems are trusted.
- Untrusted and IoT networks cannot reach the SSH service.

Password authentication should not be treated as equivalent to unrestricted Internet-facing password SSH.

Network placement materially changes exposure.

However, segmentation does not eliminate authentication risk.

### Recommended Practice

Use:

- A strong unique password.
- A dedicated administrative network or management segment.
- Firewall policy limiting source networks.
- No Internet port forwarding.
- Regular software updates.
- Log review when unexpected access attempts occur.

SSH key authentication remains preferable where automation or stronger credential protection is required.

**Validation Result:** Authentication should match the actual exposure and administrative requirements of the environment.

---

# TCP Forwarding

## `tcp_forwarding` Is Not Required for Normal SSH Access

The `Terminal & SSH` application includes an option for:

```text
tcp_forwarding
```

This is not required merely to establish a normal SSH session.

TCP forwarding is used for SSH tunneling and forwarded network connections.

Normal administrative SSH access requires:

```text
SSH port exposed
+
valid authentication
```

It does not require:

```text
tcp_forwarding: true
```

Unless SSH tunnels are specifically required, leave TCP forwarding disabled.

This follows the general security principle of enabling only the capabilities needed for the intended administrative task.

**Validation Result:** Normal SSH access functions independently of SSH TCP forwarding.

---

# Troubleshooting Reference

## Web Terminal Works but SSH Returns Connection Refused

### Symptoms

The browser-based Terminal & SSH interface works.

The following fails:

```text
ssh <configured-user>@<HAOS_IP>
```

or:

```text
Test-NetConnection <HAOS_IP> -Port 22
```

reports:

```text
PingSucceeded    : True
TcpTestSucceeded : False
```

### Cause

The Web Terminal uses Home Assistant ingress and does not require the external SSH host port to be enabled.

The SSH host port may still be blank in:

```text
Terminal & SSH
-> Network
```

### Fix

Enable:

```text
Show disabled ports
```

Set:

```text
SSH Port: 22
```

Save the Network section.

Restart the application if necessary.

### Validate

Run:

```text
Test-NetConnection <HAOS_IP> -Port 22
```

Expected:

```text
TcpTestSucceeded : True
```

---

## SSH Port Shows `22/tcp` but Connections Still Fail

### Symptom

The interface appears to show:

```text
22/tcp
```

but external SSH still fails.

### Cause

The `22/tcp` text identifies the application's internal port.

It does not mean a host-side port has been assigned.

Check the editable field beside it.

Incorrect:

```text
<blank>            22/tcp
SSH Port
```

Correct:

```text
22                 22/tcp
SSH Port
```

### Fix

Enter:

```text
22
```

into the host port field.

Save the Network section.

### Validate

```text
Test-NetConnection <HAOS_IP> -Port 22
```

---

## SSH Port Was Entered but TCP/22 Is Still Closed

### Cause

The Network section may not have been saved.

The configuration interface contains separate Save controls.

Saving Options does not necessarily save Network changes.

### Fix

Return to:

```text
Terminal & SSH
-> Network
```

Verify:

```text
SSH Port: 22
```

Then click the Save control for the Network section.

Restart the application if required.

### Validate

```text
Test-NetConnection <HAOS_IP> -Port 22
```

---

## Ping Works but SSH Does Not

Observed:

```text
PingSucceeded    : True
TcpTestSucceeded : False
```

This means:

```text
Host reachability: confirmed

SSH TCP service: not confirmed
```

Do not assume that successful ping proves that TCP/22 must work.

Likewise, do not immediately assume that a firewall is responsible.

Use the following troubleshooting order:

```text
1. Confirm HAOS responds on the network.
2. Confirm Terminal & SSH is running.
3. Confirm Web Terminal functionality.
4. Open the application's Network section.
5. Enable Show disabled ports.
6. Confirm a host port is assigned to 22/tcp.
7. Save the Network section.
8. Restart the application if required.
9. Test TCP/22 again.
10. Test SSH authentication.
```

This keeps application configuration troubleshooting separate from network-policy troubleshooting.

---

## TCP/22 Works but Authentication Fails

If:

```text
TcpTestSucceeded : True
```

but SSH authentication fails, the network and listener portions are already working.

At that point, investigate:

- SSH username.
- Password configuration.
- Authorized SSH keys.
- Client key selection.
- Application authentication settings.
- Terminal & SSH logs.

The troubleshooting boundary is:

```text
TcpTestSucceeded : False
|
+-- Listener, host-port exposure, or network path


TcpTestSucceeded : True
|
+-- TCP path is working
    |
    +-- Authentication or SSH configuration
```

**Validation Result:** Network and authentication failures should be diagnosed independently.

---

# Security and Isolation Notes

SSH is an administrative service and should only be exposed where required.

Recommended practices include:

- Restrict TCP/22 to trusted administrative networks.
- Do not expose HAOS SSH directly to the public Internet.
- Do not configure unnecessary router or firewall port forwarding.
- Keep IoT and untrusted client networks isolated from administrative SSH access.
- Use strong unique credentials.
- Prefer SSH key authentication where appropriate.
- Disable SSH forwarding unless required.
- Review application logs when unexpected authentication attempts occur.
- Keep HAOS and the `Terminal & SSH` application updated.
- Do not assume internal systems are trusted solely because they use private RFC1918 addresses.
- Test access from an actual authorized workstation after configuration changes.
- Test from an unauthorized segment when practical to confirm the service is not reachable where it should be blocked.

The preferred model is:

```text
Administrative Workstation
        |
        +-- Authorized network segment
                |
                +-- TCP/22
                        |
                        +-- HAOS Terminal & SSH


IoT / Untrusted Network
        |
        +-- TCP/22
                |
                +-- BLOCKED
```

**Validation Result:** External SSH should remain limited to the administrative paths that require it.

---

# Key Commands

## Test HAOS SSH Port from Windows

```text
Test-NetConnection <HAOS_IP> -Port 22
```

Purpose:

Tests host reachability and TCP/22 connectivity from the administrative workstation.

Important output:

```text
PingSucceeded
TcpTestSucceeded
```

Interpretation:

```text
PingSucceeded    : True
TcpTestSucceeded : False
```

means the HAOS host is reachable but SSH TCP connectivity is unavailable.

Expected after configuration:

```text
TcpTestSucceeded : True
```

---

## Connect to HAOS Terminal & SSH

General form:

```text
ssh <configured-user>@<HAOS_IP>
```

Example when `root` is configured as the SSH user:

```text
ssh root@<HAOS_IP>
```

Purpose:

Establishes an interactive SSH session through the Home Assistant `Terminal & SSH` application.

---

# Operational Validation Checklist

Use the following checklist after enabling external SSH.

- Confirm `Terminal & SSH` is running.
- Confirm the browser Web Terminal still works.
- Open the application's Network section.
- Enable `Show disabled ports` if required.
- Confirm SSH Port is set to `22`.
- Confirm the displayed mapping is effectively `22 -> 22/tcp`.
- Save the Network section independently.
- Restart `Terminal & SSH` if required.
- Run `Test-NetConnection <HAOS_IP> -Port 22`.
- Confirm `TcpTestSucceeded : True`.
- Connect with a standard SSH client.
- Confirm authentication succeeds.
- Confirm administrative commands can be executed.
- Confirm TCP/22 is not exposed to unintended network segments.
- Confirm no Internet-facing port forwarding exists for the SSH service.
- Keep `tcp_forwarding` disabled unless SSH tunnels are explicitly required.

**Validation Result:** External SSH is operational only after both the application configuration and Network port mapping are correctly saved and applied.

---

# Final State

The resulting administrative path is:

```text
Administrative Workstation
|
+-- SSH Client
    |
    +-- TCP/22
        |
        +-- <HAOS_IP>
            |
            +-- Home Assistant OS
                |
                +-- Terminal & SSH
                    |
                    +-- Authenticated shell
```

The Web Terminal remains available independently through Home Assistant:

```text
Browser
|
+-- Home Assistant
    |
    +-- Terminal & SSH
        |
        +-- Web Terminal
```

Both access methods can coexist.

The primary troubleshooting lesson is:

```text
Working Web Terminal
        |
        +-- does not prove
                |
                +-- external SSH port is enabled
```

and:

```text
Changing the SSH port
        |
        +-- does not apply until
                |
                +-- the Network section is saved
```

**Validation Result:** Direct SSH access was enabled successfully without requiring changes to the surrounding network architecture.

---

# Related Search Keywords

home-assistant, home-assistant-os, haos, terminal-and-ssh, advanced-ssh-web-terminal, ssh, external-ssh, connection-refused, tcp-22, test-netconnection, home-assistant-apps, home-assistant-addons, ingress, ssh-port, network-port-mapping, ssh-troubleshooting, home-assistant-ssh, haos-ssh

---

## Revision Control

| Version | Date | Summary | Author |
| --- | --- | --- | --- |
| **1.0.0** | 2026-09-02 | Initial documentation of enabling external SSH access to the Home Assistant OS Terminal & SSH application, including the hidden disabled-port view, separate Network Save requirement, connectivity validation, and security considerations. | projectfong |
````
