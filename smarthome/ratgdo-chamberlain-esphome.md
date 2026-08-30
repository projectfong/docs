# Chamberlain Garage Door Opener - ratgdo v2.5 ESPHome Installation Guide

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

This guide documents the physical installation of a ratgdo v2.5-compatible ESPHome controller on a compatible Chamberlain garage door opener. The installation adds local Home Assistant connectivity while retaining the existing wall control and garage-door safety sensors.

This guide focuses only on the physical ratgdo installation, wiring, initial power-up, and relevant project references.

Estimated installation time: 15-30 minutes.

---

## Hardware

### Garage Door Opener

This installation uses a Chamberlain residential garage door opener with a Security+ style terminal block.

The opener has four low-voltage terminal positions used by the wall control and obstruction sensors.

From left to right:

| Position | Terminal Color | Function                         |
| -------- | -------------- | -------------------------------- |
| 1        | Red            | Wall control                     |
| 2        | White          | Wall control/common              |
| 3        | White          | Safety sensor common             |
| 4        | Black          | Safety sensor/obstruction signal |

There are two separate white terminals. They serve different purposes, so terminal position matters during installation.

### ratgdo Controller

The controller used for this installation is a ratgdo v2.5-compatible board.

The board has three garage-door interface connections:

| ratgdo Terminal | Function                  |
| --------------- | ------------------------- |
| `WHT / GND`     | Ground/common             |
| `RED / CTRL`    | Garage-door control       |
| `BLK / OBST`    | Obstruction sensor signal |

This installation uses ESPHome firmware for Home Assistant integration.

---

## Required Tools

Only basic hand tools are needed.

* Stable stepladder
* Small wire stripper/cutter
* Small needle-nose pliers
* Small flat-blade screwdriver
* Flashlight or headlamp
* Cable ties or hook-and-loop straps for cable management
* Three-conductor connection cable
* Appropriate USB power cable and power source for the ratgdo hardware

A soldering iron, drill, crimping tool, or mains-voltage electrical tools are not normally required.

---

## Before Starting

The garage door opener must be de-energized before modifying its control wiring.

Some Chamberlain openers contain a backup battery. Unplugging the opener from AC power may therefore **not** completely remove power from the opener.

### Power Isolation

1. Place the garage door in a safe, stationary position.
2. Unplug the garage door opener from AC power.
3. If the opener has a backup battery, access the battery compartment.
4. Disconnect the black negative battery connection.
5. Disconnect the red positive battery connection if necessary to completely isolate the battery.
6. Verify that the opener and wall control no longer operate.
7. Do not modify the terminal wiring until the opener is de-energized.

Do not work on the garage-door control terminals while the opener is still operating from its backup battery.

---

## Existing Chamberlain Wiring

The existing Chamberlain wiring should remain installed.

The opener normally has wiring for:

* Wall control
* Left safety sensor
* Right safety sensor

The ratgdo is added to the appropriate opener terminals without removing the existing wall-control or safety-sensor wiring.

The finished configuration therefore retains:

* Original Chamberlain wall control
* Original left safety sensor
* Original right safety sensor
* ratgdo interface

---

## Chamberlain Terminal Identification

Before connecting the ratgdo, identify the opener terminals.

For the terminal arrangement used in this installation:

```text
LEFT                                      RIGHT

+---------+---------+---------+---------+
|   RED   |  WHITE  |  WHITE  |  BLACK  |
+---------+---------+---------+---------+
     1         2         3         4

 Wall       Wall      Safety     Safety
 Control    Common    Common     Signal
```

The two white terminals should not be treated as interchangeable.

The ratgdo ground/common connection uses the **white terminal immediately adjacent to the red terminal** for this installation.

---

## ratgdo Terminal Identification

The ratgdo enclosure identifies three connections:

```text
WHT / GND
RED / CTRL
BLK / OBST
```

The supplied three-conductor cable uses:

```text
White
Red
Black
```

For a clean installation, keep the conductor colors matched to their corresponding ratgdo functions.

---

## Wiring Map

Connect the ratgdo to the Chamberlain opener as follows:

| ratgdo Terminal | Wire  | Chamberlain Terminal          |
| --------------- | ----- | ----------------------------- |
| `RED / CTRL`    | Red   | Red                           |
| `WHT / GND`     | White | White immediately next to Red |
| `BLK / OBST`    | Black | Black                         |

Simplified:

```text
ratgdo                         Chamberlain

RED / CTRL   ---- Red -------> RED

WHT / GND    ---- White -----> WHITE
                              immediately next to RED

BLK / OBST   ---- Black -----> BLACK
```

The other Chamberlain white terminal is associated with the safety-sensor wiring and is not used for the ratgdo `WHT / GND` connection in this installation.

---

## Using the ratgdo Spring Terminals

The ratgdo board uses spring-loaded terminals with orange actuators.

The orange actuator releases the internal spring clamp.

### Insert a Wire

1. Press the orange actuator downward.
2. Keep the actuator depressed.
3. Insert the stripped conductor into the corresponding opening.
4. Release the orange actuator.
5. Gently pull on the conductor to confirm that it is securely captured.

### Remove a Wire

If a conductor is inserted into the wrong position:

1. Do not cut the wire.
2. Press the corresponding orange actuator downward.
3. Keep it depressed.
4. Gently pull the conductor out.
5. Release the actuator.

Do not pry apart the blue terminal housing.

---

## Connect the ratgdo Cable

With all power removed, connect the three-conductor cable to the ratgdo.

```text
White -> WHT / GND
Red   -> RED / CTRL
Black -> BLK / OBST
```

After inserting each conductor, gently pull on it to confirm that the spring terminal has captured it.

The conductors should not pull out under light pressure.

---

## Connect the Chamberlain Side

Connect the opposite end of the three-conductor cable to the Chamberlain terminal block.

```text
Red   -> RED

White -> WHITE immediately next to RED

Black -> BLACK
```

Leave the existing Chamberlain wiring connected.

The ratgdo connections are added alongside the existing wiring at the required terminal positions.

---

## Pre-Power Inspection

Before restoring power, inspect every connection.

Verify:

* Red ratgdo conductor is connected to `RED / CTRL`.
* White ratgdo conductor is connected to `WHT / GND`.
* Black ratgdo conductor is connected to `BLK / OBST`.
* Red conductor reaches the Chamberlain red terminal.
* White conductor reaches the Chamberlain white terminal immediately adjacent to red.
* Black conductor reaches the Chamberlain black terminal.
* Existing wall-control wiring remains connected.
* Existing safety-sensor wiring remains connected.
* No loose wire strands are visible.
* No exposed conductor can contact an adjacent terminal.
* All conductors are securely captured.
* ratgdo wiring is clear of the opener rail.
* Wiring is clear of the trolley and other moving components.

Do not restore power until the wiring has been visually inspected.

---

## Restore Power

If the garage door opener has a backup battery, reconnect it first.

Reconnect:

```text
1. Red positive
2. Black negative
```

Then reconnect AC power to the garage door opener.

Allow the opener and ratgdo to initialize.

---

## Verify Normal Garage Door Operation

Before relying on the ratgdo integration, verify that the original Chamberlain equipment continues to function normally.

Test:

1. Physical wall control.
2. Garage door opening.
3. Garage door closing.
4. Safety obstruction sensors.
5. Garage-door lighting, if applicable.

The existing wall control and obstruction sensors should continue to function after the ratgdo installation.

If the opener, wall control, or safety sensors behave unexpectedly, disconnect power and recheck the wiring before continuing.

---

## Home Assistant

For an ESPHome-flashed ratgdo, Home Assistant can communicate with the controller through the ESPHome integration after the device is connected to the appropriate Wi-Fi network.

The basic architecture is:

```text
+-----------------------------+
| Chamberlain Garage Opener   |
+-------------+---------------+
              |
              | Control / Status
              |
+-------------v---------------+
| ratgdo v2.5-compatible      |
| ESPHome                     |
+-------------+---------------+
              |
              | Local Network
              |
+-------------v---------------+
| Home Assistant              |
+-----------------------------+
```

Exact ESPHome provisioning and Home Assistant entity configuration can vary with firmware configuration and are outside the physical installation scope of this guide.

---

## Troubleshooting

### ratgdo Wire Installed in Wrong Terminal

Cause:

A conductor was inserted into the incorrect ratgdo spring terminal.

Fix:

1. Leave the equipment unpowered.
2. Press the orange actuator downward.
3. Hold it down.
4. Remove the conductor.
5. Insert it into the correct terminal.
6. Release the actuator.
7. Perform a light pull test.

Do not cut the conductor unless it is actually damaged.

### Garage Door Still Works After AC Is Unplugged

Cause:

The opener is operating from its internal backup battery.

Fix:

1. Leave AC disconnected.
2. Access the backup battery.
3. Disconnect the battery.
4. Confirm that the opener no longer operates.
5. Continue wiring only after the opener is de-energized.

### Wall Control Stops Working

Cause:

An existing wall-control conductor may have become loose while adding the ratgdo wiring.

Fix:

1. Disconnect AC power.
2. Disconnect the backup battery if installed.
3. Inspect the red wall-control terminal.
4. Inspect the adjacent white wall-control/common terminal.
5. Verify the original conductors remain securely connected.
6. Verify the ratgdo conductors are also secure.
7. Restore power and retest.

### Safety Sensors Stop Working

Cause:

An existing safety-sensor conductor may have become loose during installation.

Fix:

1. Disconnect all opener power.
2. Inspect the safety-sensor connections.
3. Verify both original sensor circuits remain connected.
4. Verify the ratgdo black obstruction connection.
5. Restore power.
6. Confirm normal safety-sensor operation before using the door.

---

## Security Notes

The ratgdo/Home Assistant configuration can provide local garage-door connectivity without requiring routine garage-door control through an external cloud service.

Recommended deployment practices include:

* Place IoT devices on an appropriate isolated network or VLAN where available.
* Permit only required communication between Home Assistant and IoT devices.
* Do not directly expose the ratgdo device to the public Internet.
* Keep ESPHome and Home Assistant authentication enabled where applicable.
* Maintain the manufacturer's physical wall control.
* Never bypass or disable the garage-door safety obstruction sensors.

This approach follows established cybersecurity frameworks and best practices, including applicable NIST SP 800-171 concepts involving network segmentation, access control, least functionality, and reduction of unnecessary external dependencies.

---

## References

### ratgdo Hardware Used

The ratgdo-compatible board used for this installation was purchased from Amazon:

https://www.amazon.com/dp/B0GNNNPFJ9

Product specifications and compatibility should be checked against the current product listing before purchasing hardware for another garage-door opener.

### ratgdo HomeKit ESP32 Project

The following project provides HomeKit firmware for supported ESP32-based ratgdo hardware:

https://github.com/ratgdo/homekit-ratgdo32

This project should not automatically be treated as firmware for every ratgdo hardware revision. Verify the supported board and microcontroller before flashing firmware.

The ESPHome-flashed hardware documented in this guide is separate from the HomeKit ESP32 firmware project referenced above.

---

## Related Search Keywords

ratgdo, garage-door-controller, ratgdo-v2-5, esphome, chamberlain, security-plus-2-0, home-assistant, local-control, smart-garage, home-automation

---

## Revision Control

| Version   | Date       | Summary                                                                                                         | Author      |
| --------- | ---------- | --------------------------------------------------------------------------------------------------------------- | ----------- |
| **1.0.0** | 2026-08-21 | Initial public-friendly ratgdo v2.5-compatible ESPHome installation guide for a Chamberlain garage door opener. | projectfong |
