# Chamberlain Garage Door Opener with ratgdo v2.5 - Quickstart

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

Install a ratgdo v2.5-compatible ESPHome controller on a compatible Chamberlain garage door opener while retaining the existing wall control and safety obstruction sensors.

This quickstart covers:

```text
Power isolation
Terminal identification
ratgdo wiring
Power restoration
Physical operation testing
Home Assistant connectivity
```

Estimated installation time:

```text
15-30 minutes
```

## Requirements

Hardware:

- Compatible Chamberlain garage door opener.
- ratgdo v2.5-compatible ESPHome controller.
- Three-conductor connection cable.
- Appropriate USB power cable and power source.

Tools:

- Stable stepladder.
- Small wire stripper/cutter.
- Small needle-nose pliers.
- Small flat-blade screwdriver.
- Flashlight or headlamp.
- Cable ties or hook-and-loop straps.

A soldering iron, drill, or mains-voltage electrical tools are not normally required.

## Safety

De-energize the garage door opener before modifying its control wiring.

Some Chamberlain openers contain a backup battery. Disconnecting AC power may not completely de-energize the opener.

Before wiring:

```text
1. Place the garage door in a safe, stationary position.
2. Disconnect AC power.
3. Disconnect the backup battery if installed.
4. Verify the opener no longer operates.
5. Modify wiring only after the opener is de-energized.
```

Do not bypass or disable the garage-door safety obstruction sensors.

## Identify the Chamberlain Terminals

The documented opener uses four low-voltage terminals:

```text
LEFT                                      RIGHT

+---------+---------+---------+---------+
|   RED   |  WHITE  |  WHITE  |  BLACK  |
+---------+---------+---------+---------+
     1         2         3         4

 Wall       Wall      Safety     Safety
 Control    Common    Common     Signal
```

The two white terminals are not interchangeable.

For this installation, ratgdo `WHT / GND` connects to the white terminal immediately adjacent to the red terminal.

Verify the actual opener terminal arrangement before making connections.

## Identify the ratgdo Terminals

The ratgdo uses:

```text
WHT / GND
RED / CTRL
BLK / OBST
```

Use matching conductor colors where practical:

```text
White -> WHT / GND
Red   -> RED / CTRL
Black -> BLK / OBST
```

## Wiring Map

Connect:

| ratgdo Terminal | Wire | Chamberlain Terminal |
| --- | --- | --- |
| `RED / CTRL` | Red | Red |
| `WHT / GND` | White | White immediately next to Red |
| `BLK / OBST` | Black | Black |

Simplified:

```text
ratgdo                         Chamberlain

RED / CTRL   ---- Red -------> RED

WHT / GND    ---- White -----> WHITE
                              immediately next to RED

BLK / OBST   ---- Black -----> BLACK
```

Leave the existing Chamberlain wall-control and safety-sensor wiring connected.

The ratgdo wiring is added alongside the existing wiring.

## Quick Installation

### 1. Remove Power

Disconnect:

```text
AC power
Backup battery, if installed
```

Verify the opener and wall control no longer operate.

### 2. Connect the ratgdo

At the ratgdo:

```text
White -> WHT / GND
Red   -> RED / CTRL
Black -> BLK / OBST
```

For each spring terminal:

```text
1. Press the orange actuator.
2. Insert the stripped conductor.
3. Release the actuator.
4. Perform a light pull test.
```

If a conductor is inserted incorrectly, press the actuator and remove it.

Do not pry apart the terminal housing.

### 3. Connect the Chamberlain Side

At the garage door opener:

```text
Red   -> RED

White -> WHITE immediately next to RED

Black -> BLACK
```

Do not remove the existing wall-control or safety-sensor wiring.

### 4. Inspect the Installation

Before restoring power, verify:

```text
[ ] Red connects RED / CTRL to RED
[ ] White connects WHT / GND to the correct WHITE terminal
[ ] Black connects BLK / OBST to BLACK
[ ] Existing wall-control wiring remains connected
[ ] Existing safety-sensor wiring remains connected
[ ] No loose wire strands are visible
[ ] No exposed conductor can contact an adjacent terminal
[ ] All conductors pass a light pull test
[ ] Wiring is clear of the opener rail
[ ] Wiring is clear of the trolley
[ ] Wiring is clear of all moving components
```

### 5. Restore Power

If a backup battery is installed, reconnect:

```text
1. Red positive
2. Black negative
```

Then reconnect AC power.

Allow the opener and ratgdo to initialize.

## Validate Garage Door Operation

Test the original Chamberlain equipment before configuring Home Assistant.

Verify:

```text
[ ] Physical wall control works
[ ] Garage door opens normally
[ ] Garage door closes normally
[ ] Safety obstruction sensors work
[ ] Garage-door lighting works, if applicable
```

If the opener, wall control, or safety sensors behave unexpectedly:

```text
1. Disconnect AC power.
2. Disconnect the backup battery.
3. Recheck all terminal wiring.
4. Correct the problem before continuing.
```

Do not rely on Home Assistant control until the original garage-door controls and safety systems operate normally.

## Home Assistant

For an ESPHome-flashed ratgdo, the basic architecture is:

```text
Chamberlain Garage Door Opener
              |
              | Control / Status
              v
     ratgdo v2.5-compatible
           ESPHome
              |
              | Local Network
              v
       Home Assistant
```

After the ratgdo is connected to the appropriate Wi-Fi network, add or verify the device through the Home Assistant ESPHome integration.

Exact ESPHome provisioning depends on the firmware configuration and is outside the physical wiring portion of this quickstart.

## Validate Home Assistant

After ESPHome integration, verify the available garage-door functions.

At minimum:

```text
[ ] ratgdo is online
[ ] Home Assistant sees the device
[ ] Door state reports correctly
[ ] Open command works
[ ] Close command works
[ ] Physical wall control still works
[ ] Safety obstruction sensors still work
```

Do not consider the installation complete until the physical safety sensors have been tested.

## Common Problems

### Garage Door Still Works After AC Is Disconnected

The opener may have a backup battery.

Disconnect the battery and verify the opener is fully de-energized before modifying wiring.

### Wall Control Stops Working

Disconnect all opener power and inspect:

```text
RED wall-control terminal
Adjacent WHITE wall-control/common terminal
Existing wall-control conductors
ratgdo conductors
```

An original wall-control conductor may have become loose while adding the ratgdo.

### Safety Sensors Stop Working

Disconnect all power and inspect:

```text
Original safety-sensor wiring
Safety common terminal
BLACK safety-sensor terminal
ratgdo BLK / OBST connection
```

Restore power only after the wiring has been corrected.

Test the safety sensors before normal door operation.

### ratgdo Wire Is in the Wrong Terminal

With the equipment unpowered:

```text
1. Press the orange actuator.
2. Hold it down.
3. Remove the conductor.
4. Insert it into the correct terminal.
5. Release the actuator.
6. Perform a light pull test.
```

Do not cut the conductor unless it is damaged.

## Security Notes

For a local Home Assistant deployment:

```text
Place the ratgdo on an appropriate IoT network or VLAN.
Permit only required communication with Home Assistant.
Do not expose the ratgdo directly to the public Internet.
Keep Home Assistant and ESPHome authentication enabled where applicable.
```

Maintain the manufacturer's physical wall control and safety obstruction sensors.

## Final Architecture

```text
Original Wall Control --------+
                              |
                              v
                    Chamberlain Opener
                              ^
                              |
Safety Sensors ---------------+
                              |
                              +---- ratgdo
                                      |
                                      | ESPHome
                                      v
                               Local Network
                                      |
                                      v
                               Home Assistant
```

The ratgdo supplements the existing Chamberlain controls.

It does not replace the physical wall control or safety obstruction sensors.

## Full Documentation

This quickstart intentionally omits the extended installation explanation, hardware purchasing references, wireless-driver-independent background, and detailed troubleshooting discussion.

See the full Chamberlain Garage Door Opener - ratgdo v2.5 ESPHome Installation Guide for additional detail.

## Related Search Keywords

ratgdo, garage-door-controller, ratgdo-v2-5, esphome, chamberlain, security-plus-2-0, home-assistant, local-control, smart-garage, home-automation

## Revision Control

| Version | Date | Summary | Author |
| --- | --- | --- | --- |
| **1.0.0** | 2026-08-31 | Initial Chamberlain ratgdo v2.5 ESPHome installation quickstart. | projectfong |
