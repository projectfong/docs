# Home Assistant OS Terminal & SSH - External SSH Quickstart

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

Enable direct SSH access to the Home Assistant OS `Terminal & SSH` application when the browser-based Web Terminal works but an external SSH client cannot connect.

The key distinction is:

```text
Web Terminal
    |
    +-- Home Assistant ingress
    |
    +-- Can work without external TCP/22


External SSH
    |
    +-- HAOS host port
    |
    +-- Must be explicitly enabled
```

A working Web Terminal does not confirm that external SSH is enabled.

## Symptoms

This quickstart applies when:

```text
[ ] Terminal & SSH is installed
[ ] Web Terminal works
[ ] HAOS is reachable on the network
[ ] External SSH cannot connect
[ ] TCP/22 is unavailable
```

Typical Windows test:

```powershell
Test-NetConnection <haos-ip> -Port 22
```

Problem state:

```text
PingSucceeded    : True
TcpTestSucceeded : False
```

This means the HAOS host is reachable, but TCP/22 is not currently reachable from the workstation.

## Enable External SSH

### 1. Open Terminal & SSH

In Home Assistant:

```text
Settings
-> Apps
-> Terminal & SSH
```

Open the application configuration.

### 2. Open the Network Section

Locate:

```text
Network
```

Enable:

```text
Show disabled ports
```

The SSH entry should become visible.

### 3. Assign the SSH Host Port

The interface shows the application's internal SSH port as:

```text
22/tcp
```

A visible `22/tcp` does not by itself mean external SSH is enabled.

Disabled:

```text
Host Port       Application Port

<blank>         22/tcp
```

Enabled:

```text
Host Port       Application Port

22              22/tcp
```

Enter:

```text
22
```

into the SSH Port host-side field.

The resulting mapping is:

```text
HAOS TCP/22
     |
     v
Terminal & SSH
     |
     v
22/tcp
```

## Save the Network Section

This step is critical.

The application configuration contains separate Save controls for different sections.

Conceptually:

```text
Terminal & SSH
|
+-- Options
|   |
|   +-- Save
|
+-- Network
    |
    +-- SSH Port
    |
    +-- Save
```

After entering:

```text
SSH Port: 22
```

click the Save control belonging to the **Network** section.

Saving another configuration section does not necessarily commit the Network port mapping.

## Restart Terminal & SSH

Restart the `Terminal & SSH` application after applying the network change.

Expected configuration:

```text
Authentication configured
        |
        v
Host TCP/22 mapped
        |
        v
Network section saved
        |
        v
Terminal & SSH restarted
```

## Validate TCP/22

From Windows:

```powershell
Test-NetConnection <haos-ip> -Port 22
```

Expected:

```text
TcpTestSucceeded : True
```

The transition should be:

```text
Before:

PingSucceeded    : True
TcpTestSucceeded : False


After:

PingSucceeded    : True
TcpTestSucceeded : True
```

If TCP succeeds, the host-port mapping and network path are working.

Authentication can then be tested separately.

## Connect with SSH

Use the username configured in `Terminal & SSH`:

```bash
ssh <configured-user>@<haos-ip>
```

If the application is configured for `root`:

```bash
ssh root@<haos-ip>
```

Enter the configured password or use the configured SSH key.

A successful connection confirms:

```text
Administrative Workstation
        |
        | TCP/22
        v
Home Assistant OS
        |
        v
Terminal & SSH
        |
        v
Authenticated Shell
```

## Validation Checklist

```text
[ ] Terminal & SSH is running
[ ] Web Terminal works
[ ] Network section opened
[ ] Show disabled ports enabled
[ ] SSH Port set to 22
[ ] Mapping shows 22 -> 22/tcp
[ ] Network section saved separately
[ ] Terminal & SSH restarted
[ ] TCP/22 test succeeds
[ ] SSH authentication succeeds
[ ] SSH is restricted to authorized networks
```

## Common Problems

### Web Terminal Works but SSH Does Not

Check:

```text
Terminal & SSH
-> Network
-> Show disabled ports
```

Verify:

```text
22 -> 22/tcp
```

A working Web Terminal does not require external TCP/22.

### `22/tcp` Is Visible but TCP/22 Is Closed

Check the host-side field.

Incorrect:

```text
<blank>         22/tcp
```

Correct:

```text
22              22/tcp
```

The `22/tcp` label identifies the application's internal port.

It does not confirm that a host port has been assigned.

### Port 22 Was Entered but Is Still Closed

Verify that the **Network section itself was saved**.

Then restart `Terminal & SSH` and test again:

```powershell
Test-NetConnection <haos-ip> -Port 22
```

### Ping Works but TCP/22 Does Not

Do not immediately assume the firewall is responsible.

Check in this order:

```text
1. HAOS network reachability
2. Terminal & SSH application status
3. Network host-port assignment
4. Network section Save
5. Application restart
6. TCP/22 connectivity
7. Firewall/network policy
```

### TCP/22 Works but SSH Login Fails

If:

```text
TcpTestSucceeded : True
```

the TCP path is already working.

Troubleshoot:

```text
SSH username
Password
SSH key
Authorized keys
Terminal & SSH authentication settings
Application logs
```

Keep transport and authentication troubleshooting separate.

## TCP Forwarding

Normal SSH access does not require:

```text
tcp_forwarding: true
```

TCP forwarding is used for SSH tunneling.

For ordinary administrative SSH:

```text
External SSH port
+
Valid authentication
=
SSH access
```

Leave TCP forwarding disabled unless SSH tunnels are specifically required.

## Security Notes

SSH is an administrative service.

Recommended configuration:

```text
Administrative Network
        |
        | TCP/22
        v
HAOS Terminal & SSH
        ^
        |
        X
IoT / Untrusted Networks
```

Use:

```text
Trusted administrative source networks
Strong unique credentials or SSH keys
Firewall restrictions
No public Internet port forwarding
Only required SSH capabilities
```

Do not expose HAOS SSH directly to the public Internet.

Password authentication may be reasonable on a tightly controlled internal administrative network, but SSH key authentication remains preferable where stronger credential protection or automation is required.

## Quick Diagnostic

```text
Web Terminal works?
        |
       YES
        |
        v
Can HAOS be reached?
        |
       YES
        |
        v
Test TCP/22
        |
        +-- FAIL
        |    |
        |    v
        |  Check Network section
        |    |
        |    v
        |  Show disabled ports
        |    |
        |    v
        |  Set 22 -> 22/tcp
        |    |
        |    v
        |  Save Network section
        |    |
        |    v
        |  Restart application
        |
        +-- PASS
             |
             v
        Test SSH authentication
```

## Key Commands

Test TCP/22 from Windows:

```powershell
Test-NetConnection <haos-ip> -Port 22
```

Connect:

```bash
ssh <configured-user>@<haos-ip>
```

## Full Documentation

This quickstart intentionally omits:

```text
Detailed troubleshooting history
Extended ingress explanation
Full connectivity analysis
Repeated TCP test interpretation
Extended authentication discussion
Detailed network-security rationale
Full operational validation procedure
```

See the full Home Assistant OS Terminal & SSH - Enabling External SSH Access documentation for the complete troubleshooting record and explanation.

## Related Search Keywords

home-assistant, home-assistant-os, haos, terminal-and-ssh, advanced-ssh-web-terminal, ssh, external-ssh, connection-refused, tcp-22, test-netconnection, ingress, ssh-port, network-port-mapping, ssh-troubleshooting

## Revision Control

| Version | Date | Summary | Author |
| --- | --- | --- | --- |
| **1.0.0** | 2026-09-02 | Initial Home Assistant OS Terminal & SSH external access quickstart. | projectfong |
