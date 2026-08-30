# TP-Link Tapo H100 Local Home Assistant Doorbell Implementation

Author: projectfong  
Copyright (c) 2026 Fong

**Status:** Operational; TP-Link ID deletion validation pending

---

## Summary

This document records the implementation of a low-cost, Home Assistant-controlled doorbell using a **GoveeLife Wireless Mini Smart Button Sensor** as the physical doorbell button and a **TP-Link Tapo H100** as the audible indoor chime.

The GoveeLife button communicates through Bluetooth and is received by the Home Assistant environment through ESP32 devices configured as ESPHome Bluetooth proxies. Home Assistant exposes the button through the **Govee Bluetooth** integration and uses the resulting event to trigger the H100 through the **TP-Link Smart Home** integration.

The implementation deliberately separates the physical button, Bluetooth transport, automation controller, audible chime, and optional video functionality rather than relying on a single proprietary video-doorbell ecosystem.

The operational path is:

    GoveeLife Wireless Mini Smart Button
    Physical designation: B5122
    Home Assistant-reported model: H5122
                     |
                     | Bluetooth
                     v
            ESP32 Bluetooth Proxy
            2 proxies deployed
                     |
                     | ESPHome
                     v
            Home Assistant OS
                     |
                     | Govee Bluetooth
                     | Button event
                     v
         Home Assistant Automation
                     |
                     | Local command
                     v
              TP-Link Tapo H100
                     |
                     | Siren action
                     | Duration: 1 second
                     v
              Audible Doorbell Chime

The Tapo H100 was successfully connected to the local IoT wireless network even though normal Tapo cloud onboarding did not complete.

During provisioning:

- The phone running the Tapo application had Internet access.
- The H100 itself did not have Internet access.
- The H100 successfully received and retained its IoT Wi-Fi configuration.
- TP-Link cloud onboarding failed.
- The Tapo application subsequently displayed zero devices.
- Home Assistant successfully connected directly to the H100 using the **TP-Link Smart Home** integration.
- Local H100 control remained operational after the H100 was unplugged, relocated, and powered back on.

A request has subsequently been submitted to permanently delete the temporary TP-Link ID used during provisioning.

Continued operation after permanent TP-Link ID deletion remains a pending validation step.

---

## Purpose

The purpose of this document is to record:

- The overall Home Assistant doorbell architecture.
- The hardware used.
- Hardware costs.
- GoveeLife wireless button integration.
- Govee Bluetooth integration behavior.
- ESPHome Bluetooth proxy transport.
- Dual ESP32 Bluetooth proxy deployment.
- Tapo H100 local integration.
- Home Assistant automation configuration.
- One-second H100 chime operation.
- Why a dedicated doorbell camera is unnecessary for this implementation.
- The H100 provisioning behavior observed when cloud access was blocked.
- The difference between phone and H100 Internet access during provisioning.
- Local H100 operation after failed Tapo onboarding.
- H100 power-cycle behavior.
- Automatic firmware update configuration.
- TP-Link ID deletion.
- Remaining post-deletion validation requirements.
- Operational recovery considerations.
- Bluetooth range and reception troubleshooting.
- Environmental placement considerations for the physical button.
- A preliminary procedure that may allow others to reproduce the configuration.

This document records observed behavior and configuration results.

It does not claim that TP-Link officially supports incomplete cloud onboarding as a deployment method.

---

## Design Objectives

The doorbell implementation was designed around several objectives:

1. Provide a simple physical doorbell button.
2. Use Home Assistant as the automation and orchestration layer.
3. Provide distributed Bluetooth reception through existing ESPHome Bluetooth proxy infrastructure.
4. Provide an audible indoor chime.
5. Avoid unnecessary duplication of functionality.
6. Avoid requiring a dedicated proprietary doorbell ecosystem.
7. Minimize ongoing vendor cloud dependencies.
8. Allow components from different manufacturers to interoperate.
9. Keep individual components independently replaceable.
10. Keep hardware cost low.
11. Maintain local operation whenever possible.

The resulting design is based on independent functional components rather than a single all-in-one doorbell appliance.

---

## Hardware

| Component | Function | Cost |
| --- | --- | ---: |
| TP-Link Tapo H100 | Indoor audible chime | $22.99 |
| GoveeLife Wireless Mini Smart Button Sensor 2-pack | Physical button inputs | $20.20 after tax |
| ESP32 Bluetooth Proxy x2 | Distributed Bluetooth reception | Existing infrastructure |
| Home Assistant OS | Automation and orchestration | Existing infrastructure |
| IoT wireless network | Device connectivity | Existing infrastructure |

### GoveeLife Button Cost

The GoveeLife buttons were purchased as a two-pack:

    Quantity:                 2
    Total cost after tax:     $20.20
    Approximate cost/button:  $10.10

### Effective Single-Doorbell Hardware Cost

Using one GoveeLife button with the Tapo H100:

    TP-Link Tapo H100:        $22.99
    One GoveeLife button:     $10.10
    --------------------------------
    Effective hardware cost:  $33.09

The ESP32 Bluetooth proxies and Home Assistant infrastructure already existed and are therefore not included in the incremental doorbell hardware cost.

### Total Purchased Doorbell Hardware

The total expenditure for the H100 and both GoveeLife buttons was:

    Tapo H100:                $22.99
    GoveeLife 2-pack:         $20.20
    --------------------------------
    Total:                    $43.19

This provides:

    1 x Tapo H100 chime
    2 x GoveeLife wireless buttons

The second button remains available as another Home Assistant event source.

---

## System Architecture

The implemented doorbell architecture is:

    +--------------------------------------+
    | GoveeLife Wireless Mini Smart Button |
    | B5122 / Home Assistant H5122         |
    +-------------------+------------------+
                        |
                        | Bluetooth
                        v
    +--------------------------------------+
    | ESP32 ESPHome Bluetooth Proxy Layer  |
    |                                      |
    | 2 Bluetooth proxies deployed         |
    +-------------------+------------------+
                        |
                        | ESPHome / Network
                        v
    +--------------------------------------+
    |          Home Assistant OS           |
    |                                      |
    | Govee Bluetooth Integration          |
    | Event Processing                     |
    | Automation Logic                     |
    | Device Coordination                  |
    | History                              |
    | Optional Notifications               |
    +-------------------+------------------+
                        |
                        | Local command
                        v
    +--------------------------------------+
    |          TP-Link Tapo H100           |
    |                                      |
    |          Audible Chime               |
    +--------------------------------------+

The GoveeLife button and Tapo H100 do not communicate directly.

The ESP32 Bluetooth proxies provide Bluetooth reception for the Home Assistant environment.

Home Assistant provides the interoperability and orchestration layer.

---

## Device Independence

An important characteristic of this implementation is that the physical input and audible output devices are from different manufacturers and use different communication paths.

    INPUT
    GoveeLife
    Wireless Mini Smart Button
              |
              | Bluetooth
              v
    ESP32 ESPHome Bluetooth Proxy
              |
              v
    HOME ASSISTANT
    Govee Bluetooth
    Automation / Orchestration
              |
              | Local network
              v
    OUTPUT
    TP-Link Tapo H100
    Chime

The GoveeLife button does not need to understand the Tapo H100.

The Tapo H100 does not need to understand the GoveeLife button.

The ESP32 Bluetooth proxies do not implement the doorbell automation logic.

Home Assistant handles the relationship between the input event and output action.

This allows each component to be selected based on its individual capabilities instead of requiring all components to belong to a single vendor ecosystem.

---

## GoveeLife Button Communication Path

The physical doorbell button is integrated into Home Assistant through the **Govee Bluetooth** integration.

Observed Home Assistant device information:

    Integration:
        Govee Bluetooth

    Device:
        Doorbell

    Model reported by Home Assistant:
        H5122

    Physical product designation:
        B5122

    Entities:
        3

Home Assistant therefore reports the button as model **H5122**, while the physical product designation is **B5122**.

Both identifiers are retained in this documentation because they may be encountered when identifying, purchasing, configuring, or troubleshooting the device.

### Bluetooth Transport

Bluetooth reception for the Home Assistant environment is provided by two ESP32 devices configured as ESPHome Bluetooth proxies.

The resulting input path is:

    GoveeLife Wireless Mini Smart Button
                  |
                  | Bluetooth
                  v
           ESP32 Bluetooth Proxy
                  |
                  | ESPHome
                  | Network transport
                  v
          Home Assistant OS
                  |
                  | Govee Bluetooth
                  v
             Button Event
                  |
                  v
         Doorbell Automation

The physical button therefore does not require a direct Wi-Fi connection for the documented Home Assistant event path.

The button also does not communicate directly with the Tapo H100.

Home Assistant receives the Bluetooth-derived button event through the available Bluetooth proxy infrastructure and independently issues the appropriate command to the H100.

### Bluetooth Proxy Deployment

The Home Assistant environment currently has:

    ESP32 Bluetooth proxies: 2

The proxies extend Bluetooth reception beyond reliance on a Bluetooth adapter located directly at the Home Assistant host.

The existence of two proxies should not be interpreted as verified redundant reception for the doorbell.

The current implementation establishes that ESPHome Bluetooth proxy infrastructure is available and that the GoveeLife button is successfully received by Home Assistant.

Reception through each individual proxy has not been independently documented as a doorbell redundancy test.

---

## Doorbell Event Flow

The complete event sequence is:

    Visitor presses button
              |
              v
    GoveeLife button transmits
    Bluetooth event
              |
              v
    ESP32 Bluetooth proxy receives
    Bluetooth communication
              |
              v
    ESPHome provides Bluetooth
    transport to Home Assistant
              |
              v
    Govee Bluetooth integration
    exposes device event
              |
              v
    Home Assistant receives event
              |
              v
    Doorbell automation executes
              |
              v
    Home Assistant OS activates
    Tapo H100 siren
              |
              v
    H100 plays selected doorbell tone
              |
              v
    Siren automatically stops
    after 1 second

Home Assistant remains responsible for determining what should happen when the physical button is pressed.

---

## GoveeLife Wireless Button

The physical doorbell input is a **GoveeLife Wireless Mini Smart Button Sensor**.

The physical product designation is:

    B5122

Home Assistant identifies the device through the Govee Bluetooth integration as:

    H5122

The button appears in Home Assistant as an event-capable device.

The doorbell automation uses:

    Event received
        |
        +-- Button 1

The button press acts as the trigger for the automation.

Conceptually:

    WHEN
        GoveeLife Button 1 is pressed

    THEN
        Activate Tapo H100 chime

This allows the inexpensive GoveeLife button to function as the physical doorbell button without requiring a matching Govee or Tapo chime.

---

## Home Assistant Doorbell Automation

The Home Assistant automation consists of:

### Trigger

    Event received
        |
        +-- Button 1

### Conditions

No conditions are currently required for the basic doorbell operation.

    Conditions: None

### Primary Action

    Turn on siren

    Entity:
        Tapo Chime

    Duration:
        1 second

The resulting automation can be represented as:

    WHEN
        GoveeLife Button 1 event received

    AND IF
        No additional conditions

    THEN
        Turn on siren
            Entity: Tapo Chime
            Duration: 1 second

The exact Home Assistant automation YAML has not been included because the current documented evidence establishes the UI-configured trigger and action but does not provide an exported automation YAML configuration.

No assumed event type, entity ID, device ID, or automation identifier is documented.

---

## Tapo H100 Siren Configuration

Home Assistant exposes the H100 as a siren-capable entity.

The action provides configurable properties including:

    Tone
    Volume
    Duration

The current doorbell implementation explicitly uses:

    Target:
        Tapo Chime

    Duration:
        1 second

This prevents the H100 from operating as a continuous alarm when used as the doorbell chime.

Instead:

    Button press
         |
         v
    H100 activated
         |
         v
    Doorbell tone
         |
         v
    1 second
         |
         v
    H100 stops

---

## Alarm Sound Selection

Home Assistant also exposes the H100 alarm sound selection.

Observed options include:

    Doorbell Ring 1
    Doorbell Ring 2

Home Assistant successfully changed the H100 alarm sound.

Observed Home Assistant history included:

    4:48:18 PM - Doorbell Ring 2
    4:48:27 PM - Doorbell Ring 1

This provides additional confirmation that Home Assistant can locally control H100 configuration properties.

---

## Optional Mobile Notification

A mobile notification action exists within the doorbell automation configuration but is currently disabled.

The primary doorbell function does not depend on mobile notification delivery.

The operational path remains:

    Button
      |
      v
    Bluetooth
      |
      v
    ESP32 Proxy
      |
      v
    Home Assistant OS
      |
      v
    H100
      |
      v
    Chime

This keeps the physical doorbell functional independently of whether a mobile notification is enabled.

---

## Potential Future Automation Actions

Because Home Assistant receives the original button event, a doorbell press can trigger more than an audible chime.

The same event could eventually produce:

    GoveeLife button press
              |
              v
        Home Assistant
              |
              +-- Ring Tapo H100
              |
              +-- Send mobile notification
              |
              +-- Reference a video source
              |
              +-- Trigger lighting
              |
              +-- Apply presence conditions
              |
              +-- Apply time-of-day logic
              |
              +-- Record event history
              |
              +-- Trigger other security workflows

The GoveeLife button should therefore be considered a generic Home Assistant event source rather than a dedicated proprietary doorbell transmitter.

---

## Video Camera Not Required

A dedicated doorbell camera is intentionally not part of this implementation.

The purpose of this system is to provide a reliable physical doorbell trigger and audible chime through Home Assistant:

    GoveeLife Button
           |
           v
    Bluetooth Proxy
           |
           v
    Home Assistant
           |
           v
    Tapo H100 Chime

Video functionality is outside the scope of this implementation.

A dedicated camera is not required for the documented doorbell architecture and can be integrated independently if needed.

The doorbell implementation remains focused on:

- Physical visitor input.
- Bluetooth event transport.
- Local Home Assistant event processing.
- Audible indoor notification.
- Optional Home Assistant notifications.
- Optional automation actions.

If video information is required by a future automation, Home Assistant can integrate an appropriate video source independently of the doorbell button and chime.

### Separation of Functions

The doorbell does not need to be a single device containing every possible function.

The implementation deliberately separates:

    Physical Button
          |
          v
    Bluetooth Transport
          |
          v
    Automation
          |
          v
    Audible Notification
          |
          +---- Optional independent video source

This avoids duplicating capabilities solely because commercial video doorbells commonly bundle them together.

---

## Desired H100 Network Architecture

The desired steady-state configuration is:

    Home Assistant
          |
          | Local network communication
          |
          v
    Tapo H100
          |
          v
       Chime

The desired operational path is not:

    Home Assistant
          |
          v
    TP-Link Cloud
          |
          v
    Tapo H100

The H100 only needs to receive local commands from Home Assistant and activate its chime.

---

## Initial Tapo H100 Provisioning

The H100 presented an unexpected but useful provisioning behavior.

Normal Tapo onboarding did not successfully complete.

However, enough of the provisioning process completed for the H100 to:

- Receive the IoT wireless configuration.
- Join the IoT network.
- Obtain local network connectivity.
- Remain accessible to Home Assistant.
- Retain its configuration across a power cycle.

---

## Temporary Tapo Application Installation

The Tapo application was installed temporarily on a phone for initial H100 provisioning.

A TP-Link ID was also created because the Tapo application required account authentication.

The intention was not to use the Tapo application as the long-term operational interface.

Home Assistant was intended to become the operational controller after provisioning.

---

## Important Network Distinction

During H100 provisioning, the phone and H100 were connected through the IoT environment but had different Internet permissions.

The phone could access the Internet.

The H100 could not.

Observed behavior:

    Phone
      |
      +-- IoT Wi-Fi
      |
      +-- Internet access: ALLOWED
      |
      +-- TP-Link authentication: WORKING


    Tapo H100
      |
      +-- IoT Wi-Fi
      |
      +-- Local network access: WORKING
      |
      +-- Internet access: BLOCKED
      |
      +-- TP-Link cloud onboarding: FAILED

This difference appears to explain the unusual provisioning result.

---

## Provisioning Sequence

### Step 1 - Authenticate Tapo Application

The phone running the Tapo application had Internet access.

This allowed:

    Phone
      |
      v
    Internet
      |
      v
    TP-Link

to function normally.

The Tapo application could therefore authenticate using the temporary TP-Link ID.

### Step 2 - Begin H100 Provisioning

H100 setup was initiated from the Tapo application.

The H100 received sufficient configuration information to join the IoT wireless network.

### Step 3 - H100 Joins IoT Wi-Fi

The H100 successfully associated with the intended 2.4 GHz IoT wireless network.

The device became locally reachable.

### Step 4 - H100 Attempts Cloud Onboarding

The H100 itself did not have Internet access.

The resulting network behavior was effectively:

    Phone -> IoT network -> Internet -> TP-Link
                                  |
                                  +-- WORKING


    H100 -> IoT network -> Internet -> TP-Link
                                  |
                                  +-- BLOCKED

The Tapo onboarding process therefore did not successfully complete.

### Step 5 - Tapo Application Reports No Device

After the failed onboarding attempt, the Tapo application showed:

    Devices: 0

The H100 did not appear as a managed device in the application.

### Step 6 - H100 Remains Locally Reachable

Despite the failed cloud onboarding process, the H100 remained connected to the IoT wireless network.

This allowed Home Assistant integration to proceed.

---

## Observed Provisioning Result

The observed result was:

    Wi-Fi provisioning:       SUCCESS
    Local network access:     SUCCESS
    TP-Link cloud onboarding: FAILED
    Tapo application binding: NOT OBSERVED
    Home Assistant access:    SUCCESS

This distinction is central to the implementation.

---

## Adding the H100 to Home Assistant

### Step 1 - Open Integrations

In Home Assistant:

    Settings
      -> Devices & services
      -> Add Integration

Searching for:

    Tapo

displayed multiple TP-Link-related options.

The integration used was:

    TP-Link Smart Home

---

## TP-Link Smart Home Integration

The H100 was added through Home Assistant's **TP-Link Smart Home** integration.

The configuration dialog provided:

    Host

with the description:

    Hostname or IP address of your TP-Link device.

The H100 IP address can be entered directly.

Example:

    192.168.x.x

Do not include:

    http://

Using the device IP address directly is particularly useful when Home Assistant and IoT devices are separated by network boundaries that prevent automatic discovery.

---

## Local H100 Authentication

Home Assistant successfully authenticated to the H100 and created the device integration.

The H100 subsequently appeared in Home Assistant and exposed usable entities.

At this stage:

    Tapo application:
        H100 not listed

    Home Assistant:
        H100 available

Home Assistant could control the H100 even though normal Tapo application onboarding had not completed.

The exact authentication mechanism responsible for continued local access has not been independently established and is therefore not assumed in this document.

---

## Verified Home Assistant H100 Functionality

The following H100 functionality has been directly observed:

- H100 appears in Home Assistant.
- Alarm sound selection is exposed.
- Alarm sound can be changed.
- H100 siren functionality is exposed.
- Siren duration can be configured.
- H100 can function as the doorbell chime.
- Home Assistant records H100 state changes.
- H100 reconnects after loss of power.
- Home Assistant regains communication after the H100 restarts.

---

## H100 Power-Cycle Validation

After successful Home Assistant integration, the H100 was:

1. Unplugged.
2. Physically moved to its intended permanent location.
3. Plugged back into power.
4. Allowed to restart.
5. Allowed to reconnect to the IoT network.

The H100 successfully reconnected.

Home Assistant also regained communication with the H100.

No successful Tapo application onboarding was required after the power cycle.

### Verified Power-Cycle Path

    H100 power removed
            |
            v
    H100 physically relocated
            |
            v
    H100 powered on
            |
            v
    Reconnects to IoT Wi-Fi
            |
            v
    Home Assistant OS reconnects to H100
            |
            v
    Local H100 control continues

This confirms that the H100 retained its wireless configuration and sufficient local configuration to resume operation after complete power loss.

---

## H100 Internet Access

The H100 was provisioned in an environment where it could communicate locally but could not reach the Internet.

This is important because the intended permanent configuration does not require the H100 to communicate with TP-Link cloud services during normal operation.

The desired policy is:

    H100 -> Local network: ALLOW
    H100 -> Required Home Assistant path: ALLOW
    H100 -> Internet/WAN: BLOCK

The firewall should not be opened broadly merely to satisfy unnecessary cloud connectivity.

---

## Firmware Update Policy

Automatic firmware updates were disabled on the H100.

The device currently performs the required function:

    Home Assistant command
             |
             v
            H100
             |
             v
            Ring

There is therefore no operational requirement to automatically install firmware merely because a newer version becomes available.

Firmware changes could potentially alter:

- Local protocol behavior.
- Authentication behavior.
- Home Assistant compatibility.
- Device network requirements.
- Cloud requirements.
- H100 onboarding behavior.
- Entity behavior.

### Firmware Update Criteria

Firmware should be updated intentionally when there is a specific reason, such as:

- A relevant security vulnerability.
- A Home Assistant compatibility requirement.
- A bug affecting the deployment.
- Required support for new functionality.
- A known operational problem resolved by the update.

Before updating, record:

    Current firmware version
    Target firmware version
    Release notes
    Reason for update
    Home Assistant compatibility status

After updating, repeat the local-operation validation tests.

---

## Known-Good Configuration Principle

Once a device performs its required function reliably, the current configuration should be considered a known-good operational state.

For this H100:

    Required function:
        Ring when instructed by Home Assistant

    Current result:
        Working

Unnecessary configuration or firmware changes should therefore be avoided unless they provide a clear benefit.

---

## Tapo Application State

After provisioning failed, the Tapo application displayed no devices.

Observed state:

    Tapo Application
    ----------------
    Devices: 0

At the same time:

    Home Assistant
    --------------
    H100: Available
    Local control: Working

This difference is significant.

The H100 was locally functional despite not appearing as a managed device in the Tapo application.

---

## TP-Link Product Registration State

The TP-Link Product Registration System was also checked.

Observed state:

    Total registered products: 0

No products appeared in the account's product-registration interface.

Product registration and Tapo cloud device binding should not automatically be considered the same backend mechanism.

This observation is therefore supporting information only.

It should not independently be interpreted as proof that no TP-Link backend record of the H100 exists.

---

## TP-Link ID Deletion

The TP-Link ID was created solely to satisfy the initial provisioning workflow.

Because the Tapo application showed no devices and the H100 was already functioning locally through Home Assistant, permanent account deletion was initiated.

---

## TP-Link Account Deletion Prerequisites

TP-Link's account-deletion interface displayed the following prerequisites:

    No devices bound to your TP-Link ID

    No subscriptions connected to your TP-Link ID

The deletion interface also warned that:

- Account data will be cleared.
- Account data cannot be restored.
- The account cannot subsequently be used to log in.
- Associated TP-Link cloud services will no longer be available.
- Associated posts and account content will be removed.

---

## Account Deletion Survey

During deletion, TP-Link requested a reason for leaving.

Available options included:

    I'm concerned about my data privacy.

    My email account has been stolen or hacked.
    I want to delete it to protect my data.

    I have another account.

    I prefer not to say.

    Other

The selected response was:

    I prefer not to say.

No additional reason was provided.

---

## TP-Link ID Deletion Request

The permanent deletion request was successfully submitted.

TP-Link displayed:

    Your Request Has Been Submitted

and:

    An email will be sent to you within 72 hours to notify
    whether your TP-Link ID has been deleted successfully or not.

### Current Status

    TP-Link ID deletion request: SUBMITTED
    Final deletion confirmation: PENDING
    Expected response window:    Within 72 hours

The account must not yet be documented as successfully deleted.

---

## Required Post-Deletion Validation

Once TP-Link confirms that the TP-Link ID has been permanently deleted, additional testing is required.

These tests determine whether the account has any ongoing operational relationship with the locally integrated H100.

---

## Test 1 - Basic Doorbell Operation

Press the GoveeLife doorbell button.

Expected sequence:

    GoveeLife button
           |
           v
    Bluetooth transmission
           |
           v
    ESP32 Bluetooth proxy
           |
           v
    Govee Bluetooth / Home Assistant OS
           |
           v
    Doorbell automation
           |
           v
    H100 chime

Expected result:

    PASS

The H100 should ring normally.

---

## Test 2 - Direct Home Assistant H100 Control

From Home Assistant, directly activate the H100 siren.

Expected result:

    Home Assistant -> H100 -> audible chime

Record:

    PASS

or:

    FAIL

---

## Test 3 - Entity Control

Change an exposed H100 property such as:

    Doorbell Ring 1
            |
            v
    Doorbell Ring 2

Expected result:

Home Assistant successfully changes the H100 property without requiring TP-Link cloud connectivity.

---

## Test 4 - Confirm WAN Remains Blocked

Verify that the H100 still cannot initiate Internet connections.

Desired state:

    H100 -> Local network: ALLOW
    H100 -> Home Assistant: ALLOW
    H100 -> Internet: BLOCK

The test should not temporarily restore Internet access.

---

## Test 5 - H100 Power Cycle After Account Deletion

Disconnect power from the H100.

Wait several seconds.

Restore power.

Expected sequence:

    H100 boots
        |
        v
    H100 reconnects to IoT Wi-Fi
        |
        v
    Home Assistant OS reconnects to H100

---

## Test 6 - Doorbell After H100 Power Cycle

After the H100 reconnects, press the GoveeLife button.

Expected sequence:

    Button press
        |
        v
    Bluetooth proxy
        |
        v
    Home Assistant OS
        |
        v
    H100
        |
        v
    Chime

This is one of the most important post-deletion tests.

It demonstrates whether the H100 can recover from complete power loss without TP-Link account or Internet access.

---

## Test 7 - TP-Link Integration Reload

Reload the Home Assistant TP-Link Smart Home integration.

Expected sequence:

    Integration reload
           |
           v
    Local H100 authentication
           |
           v
    H100 available
           |
           v
    Doorbell works

This helps determine whether operation depends on an existing active Home Assistant session.

---

## Test 8 - Home Assistant OS Restart

Restart Home Assistant OS or Home Assistant Core as appropriate.

After Home Assistant returns:

1. Verify the H100 becomes available.
2. Verify the ESPHome Bluetooth proxy infrastructure becomes available.
3. Verify the GoveeLife button remains available.
4. Press the doorbell button.
5. Confirm the H100 rings.
6. Confirm the H100 still has no Internet access.

Expected result:

    Home Assistant OS restart
            |
            v
    Integrations reload
            |
            +-- ESPHome Bluetooth proxies available
            |
            +-- GoveeLife available
            |
            +-- H100 available
            |
            v
    Button press
            |
            v
    Doorbell rings

---

## Current Validation Results

| Test | Result |
| --- | --- |
| ESP32 Bluetooth proxies deployed | PASS - 2 |
| Govee Bluetooth integration active | PASS |
| Govee button identified by Home Assistant as H5122 | PASS |
| Govee button visible in Home Assistant OS | PASS |
| Govee button event received by Home Assistant | PASS |
| Button event usable as automation trigger | PASS |
| Govee button direct Wi-Fi required | NO |
| Govee button weatherproof capability tested | NOT TESTED |
| H100 joined IoT Wi-Fi | PASS |
| Phone had Internet during H100 provisioning | PASS |
| H100 Internet access blocked during onboarding | PASS |
| Tapo onboarding completed | NO |
| H100 visible in Tapo app | NO |
| Tapo app device count | 0 |
| H100 added to Home Assistant | PASS |
| Home Assistant local H100 control | PASS |
| H100 alarm sound control | PASS |
| H100 siren action available | PASS |
| H100 siren duration configurable | PASS |
| Doorbell configured for 1-second H100 activation | PASS |
| Govee button triggers H100 through Home Assistant | PASS |
| H100 survives complete power cycle | PASS |
| Home Assistant reconnects after H100 power cycle | PASS |
| Individual proxy redundancy for doorbell independently tested | NOT VERIFIED |
| TP-Link product registration count | 0 |
| Automatic H100 firmware updates disabled | PASS |
| TP-Link ID deletion requested | PASS |
| TP-Link ID deletion completed | PENDING |
| Home Assistant works after TP-Link ID deletion | PENDING |
| Doorbell works after account deletion | PENDING |
| H100 power cycle after account deletion | PENDING |
| Home Assistant integration reload after account deletion | PENDING |
| Home Assistant OS restart after account deletion | PENDING |

---

## What Is Verified

The following statements are supported by direct observation during this implementation:

1. The GoveeLife wireless button is available through Home Assistant's Govee Bluetooth integration.
2. Home Assistant reports the doorbell device as model H5122.
3. The physical product designation is B5122.
4. Two ESP32 devices are deployed as ESPHome Bluetooth proxies.
5. The GoveeLife button can function as a Home Assistant event source.
6. A GoveeLife button press can trigger a Home Assistant automation.
7. Home Assistant can use that event to activate the Tapo H100.
8. The H100 exposes siren functionality in Home Assistant.
9. The H100 siren can be configured with a one-second duration.
10. The GoveeLife and Tapo devices do not need to belong to the same vendor ecosystem.
11. The H100 can receive its IoT Wi-Fi configuration even when complete Tapo onboarding does not finish.
12. The H100 remained connected to the local IoT network after failed Tapo onboarding.
13. The Tapo application showed zero devices afterward.
14. Home Assistant successfully added and controlled the H100.
15. Home Assistant exposed H100 entities including alarm sound selection.
16. The H100 remained operational after being unplugged, physically moved, and powered back on.
17. Home Assistant regained communication with the H100 after the H100 power cycle.
18. The tested Home Assistant functions did not require successful Tapo application onboarding.
19. A TP-Link ID deletion request was successfully submitted.
20. Automatic H100 firmware updates can be disabled while retaining the current working configuration.
21. The installed GoveeLife button is located in a protected area where direct exposure to most weather conditions is not expected to be an operational concern.

---

## What Is Not Yet Verified

The following must not currently be stated as confirmed.

### Bluetooth Proxy Redundancy

Two ESP32 Bluetooth proxies are deployed in the Home Assistant environment.

It has not been independently documented that the doorbell remains reachable through each proxy individually.

The presence of two proxies therefore does not by itself establish redundant Bluetooth coverage for this specific button.

### GoveeLife Button Weather Resistance

The GoveeLife H5122 / B5122 has not been tested in this deployment for weatherproof or water-resistant operation.

No weatherproof capability is assumed as part of this implementation.

The protected installation location should not be interpreted as evidence that the device itself is weatherproof, waterproof, or outdoor-rated.

### Accountless Long-Term Operation

It is not yet verified that the H100 will continue operating indefinitely after the TP-Link ID has been permanently deleted.

### Authentication Implementation

The exact authentication information retained by the H100 has not been independently inspected.

Possible mechanisms might include:

    Cached credentials
    Derived authentication keys
    Local tokens
    Persistent authentication state
    Device-specific keys

These are possible implementation mechanisms only.

They are not established facts in this deployment.

### Behavior After Account Deletion

Until TP-Link confirms account deletion and the validation tests are completed, it is unknown whether deleting the TP-Link ID will affect future Home Assistant authentication.

### Future Firmware Behavior

It is not verified that future H100 firmware versions will retain the same behavior.

### Factory Reset Behavior

The current working H100 has not been factory-reset after establishing the configuration.

A factory reset may require repeating the provisioning process.

### Reproducibility

This behavior has currently been observed on one deployment.

It has not yet been reproduced across multiple factory-reset H100 units.

---

## Observed End-to-End Model

Based strictly on observed behavior:

                    +---------------------+
                    |  GoveeLife Button   |
                    | B5122 / Home        |
                    | Assistant H5122      |
                    +----------+----------+
                               |
                               | Bluetooth
                               v
                    +---------------------+
                    | ESP32 Bluetooth     |
                    | Proxy Infrastructure|
                    | 2 proxies deployed  |
                    +----------+----------+
                               |
                               | ESPHome
                               v
                    +---------------------+
                    | Home Assistant OS   |
                    | Govee Bluetooth     |
                    | Automation          |
                    +----------+----------+
                               |
                               | Local command
                               v
                    +---------------------+
                    |      Tapo H100      |
                    |       Chime         |
                    +---------------------+


                    +---------------------+
                    |        Phone        |
                    |      Tapo App       |
                    +----------+----------+
                               |
                               | Internet allowed
                               v
                    +---------------------+
                    |       TP-Link       |
                    |    Cloud Services   |
                    +---------------------+


                    +---------------------+
                    |      Tapo H100      |
                    +----------+----------+
                               |
                               | IoT Wi-Fi
                               v
                    +---------------------+
                    |     IoT Network     |
                    +----------+----------+
                               |
                  +------------+-------------+
                  |                          |
                  v                          v
           Local network                  Internet
              ALLOWED                      BLOCKED
                  |                          |
                  v                          X
         +--------------------+
         | Home Assistant OS  |
         +--------------------+

The important observed results are:

    Govee Bluetooth input:      WORKING
    ESPHome proxy path:         WORKING
    Home Assistant automation:  WORKING
    Cloud onboarding:           FAILED
    Local H100 operation:       WORKING

---

## Preliminary Reproduction Procedure

This section remains preliminary until TP-Link account deletion and all post-deletion validation tests are complete.

### Requirements

- TP-Link Tapo H100.
- GoveeLife Wireless Mini Smart Button Sensor B5122.
- Home Assistant OS.
- Govee Bluetooth integration.
- Bluetooth reception available to Home Assistant.
- ESP32 configured as an ESPHome Bluetooth proxy for the documented architecture.
- TP-Link Smart Home integration.
- 2.4 GHz IoT wireless network.
- Firewall capable of controlling H100 WAN access.
- Phone capable of accessing the IoT network.
- Temporary Tapo application installation.
- Temporary TP-Link ID.
- A suitable protected installation location if the button is installed near an exterior entrance.

The documented deployment uses two ESP32 Bluetooth proxies.

A reproducing environment does not necessarily require exactly two proxies, but sufficient Bluetooth reception between the button and Home Assistant is required.

The documented deployment does not establish that the GoveeLife button is suitable for unprotected outdoor exposure.

### Preliminary GoveeLife Button Procedure

1. Configure ESP32 Bluetooth proxy infrastructure for Home Assistant.
2. Verify the ESPHome Bluetooth proxy is available to Home Assistant.
3. Place the GoveeLife B5122 within usable Bluetooth reception range.
4. If the button is installed near an exterior entrance, select a protected location that minimizes direct weather exposure.
5. Open Home Assistant.
6. Verify the **Govee Bluetooth** integration detects or exposes the device.
7. Verify Home Assistant identifies the device.
8. Verify the button appears as an event-capable device.
9. Press the button.
10. Verify Home Assistant receives the button event.
11. Confirm the event can be selected as an automation trigger.

Observed device information in the documented deployment:

    Integration:                      Govee Bluetooth
    Home Assistant-reported model:    H5122
    Physical designation:             B5122
    Device role:                      Doorbell
    Entities:                         3

### Preliminary H100 Procedure

1. Create a temporary TP-Link ID.
2. Install the Tapo application.
3. Allow the phone to access the Internet.
4. Begin H100 provisioning.
5. Allow the H100 to receive the IoT Wi-Fi configuration.
6. Prevent the H100 itself from reaching the Internet.
7. Allow the H100 to associate with the IoT wireless network.
8. Allow the Tapo cloud onboarding process to fail.
9. Verify the H100 has obtained an IP address.
10. Verify the Tapo application does not show the H100 as successfully added.
11. Open Home Assistant.
12. Add the TP-Link Smart Home integration.
13. Enter the H100 IP address if automatic discovery does not find it.
14. Complete Home Assistant's local H100 integration.
15. Verify the H100 appears in Home Assistant.
16. Activate the H100 chime from Home Assistant.
17. Power-cycle the H100.
18. Verify the H100 reconnects.
19. Verify Home Assistant reconnects.
20. Verify the H100 chime still works.
21. Verify the Tapo application still contains no bound devices.
22. Request deletion of the temporary TP-Link ID.
23. Wait for confirmed account deletion.
24. Complete all post-deletion validation tests.

### Warning

This procedure currently represents an observed configuration, not a fully validated universal H100 provisioning method.

Firmware, hardware revision, Home Assistant version, ESPHome version, TP-Link integration changes, Govee Bluetooth integration changes, or Tapo application changes could alter the behavior.

---

## Complete Doorbell Reproduction

After establishing the Bluetooth input path and H100 connectivity:

1. Verify an ESPHome Bluetooth proxy is available to Home Assistant.
2. Verify the GoveeLife button appears through the Govee Bluetooth integration.
3. Verify Home Assistant receives the button event.
4. Verify the H100 is available through the TP-Link Smart Home integration.
5. Create a new automation.
6. Configure the GoveeLife button event as the trigger.
7. Add the H100 **Turn on siren** action.
8. Select the **Tapo Chime** entity.
9. Enable the **Duration** property.
10. Set the duration to:

        1

11. Save the automation.
12. Press the GoveeLife button.
13. Confirm the H100 produces the selected doorbell tone.
14. Review the automation trace if the chime does not activate.

The complete logical path should be:

    B5122 / H5122
          |
          v
    Bluetooth
          |
          v
    ESP32 Proxy
          |
          v
    Govee Bluetooth
          |
          v
    Home Assistant Automation
          |
          v
    TP-Link Smart Home
          |
          v
    H100 Chime

---

## Recommended Steady-State Configuration

If all post-account-deletion tests pass, the preferred configuration is:

    GoveeLife button transport:          Bluetooth
    Govee Bluetooth integration:        ACTIVE
    ESP32 Bluetooth proxies:            2 DEPLOYED
    GoveeLife button location:           PROTECTED AREA
    Weatherproof capability:             NOT TESTED
    Tapo application installed:         NO
    TP-Link ID operationally needed:    NO
    H100 Internet access:               BLOCKED
    H100 automatic firmware updates:    DISABLED
    H100 local network access:          ALLOWED
    Home Assistant access to H100:      ALLOWED
    GoveeLife button available to
    Home Assistant:                     YES
    Home Assistant controls
    doorbell logic:                     YES
    H100 provides audible output:       YES
    Dedicated doorbell camera:          NOT REQUIRED

The `TP-Link ID operationally needed` value remains pending until account deletion is confirmed and post-deletion testing succeeds.

---

## Logical Firewall Policy

The exact implementation depends on the network architecture.

The desired logical policy is:

    SOURCE:
        Home Assistant

    DESTINATION:
        H100

    ACTION:
        ALLOW

    PURPOSE:
        Local Home Assistant control

For H100 Internet access:

    SOURCE:
        H100

    DESTINATION:
        WAN / Internet

    ACTION:
        BLOCK

    PURPOSE:
        Prevent unnecessary cloud communication

Any required local return traffic must remain permitted.

Do not broadly permit unnecessary inter-network or Internet access solely to make the H100 operational.

---

## Operational Recovery

### If the H100 Stops Responding

Check:

1. H100 power.
2. H100 IP address.
3. IoT wireless association.
4. DHCP lease.
5. Home Assistant OS routing to the IoT network.
6. Firewall rules.
7. TP-Link Smart Home integration status.
8. H100 entity availability.
9. Home Assistant logs.

Do not immediately restore H100 Internet access.

First determine whether the problem is local.

---

## If the Doorbell Button Stops Working

Determine whether the failure is on the Bluetooth input side, automation layer, or H100 output side.

### H100 Direct Test

Test the H100 directly from Home Assistant:

    Home Assistant
          |
          v
    Turn on Tapo Chime siren

If the H100 rings:

    H100 path: WORKING

Then investigate the input path:

    GoveeLife Button
          |
          v
    Bluetooth
          |
          v
    ESP32 Proxy
          |
          v
    Govee Bluetooth
          |
          v
    Home Assistant Event

Check:

- GoveeLife button battery.
- Govee Bluetooth device availability.
- ESPHome Bluetooth proxy availability.
- ESP32 proxy network connectivity.
- Button event reception.
- Automation trigger.
- Automation trace.

If the H100 does not ring when directly activated, investigate the H100 integration and network path instead.

This separation makes troubleshooting considerably easier than treating the entire doorbell as one monolithic device.

---

## If Bluetooth Input Stops Working

Check the input path independently of the H100:

1. Verify the GoveeLife button has power.
2. Verify the Govee Bluetooth integration is loaded.
3. Verify the H5122 device remains available in Home Assistant.
4. Verify the ESP32 Bluetooth proxies are online.
5. Verify ESPHome connectivity to Home Assistant.
6. Press the button.
7. Observe whether Home Assistant receives the event.
8. Review Home Assistant and ESPHome logs if the event is not received.

Do not troubleshoot the H100 until the button event has first been confirmed.

This isolates:

    INPUT FAILURE

from:

    OUTPUT FAILURE

### Bluetooth Range and Reception Check

If button events become intermittent, verify the Bluetooth reception path before troubleshooting the automation or H100.

Check:

1. Verify both ESP32 Bluetooth proxies remain online.
2. Verify the Govee H5122 remains available through the Govee Bluetooth integration.
3. Press the button several times from its installed location.
4. Confirm Home Assistant consistently receives the button event.
5. Review available Bluetooth scanner or proxy diagnostic information for reception quality.
6. If RSSI information is exposed for the device or scanner in the current Home Assistant configuration, use it as an additional indication of signal quality.
7. Test the button closer to a Bluetooth proxy to determine whether physical range is contributing to missed events.

A weak or intermittent Bluetooth path can produce an input-side failure even while the Home Assistant automation and Tapo H100 output path remain fully operational.

---

## If Home Assistant Loses H100 Authentication

If TP-Link account deletion has completed and Home Assistant subsequently loses H100 authentication, record the failure before changing the configuration.

Capture:

    Date and time
    Home Assistant error
    TP-Link integration error
    H100 availability state
    H100 IP address
    Firewall state
    Whether Home Assistant OS was restarted
    Whether H100 was restarted
    Last known successful operation

Do not factory-reset the H100 until the failure has been documented.

This information may determine whether the deleted TP-Link ID had an unexpected long-term relationship with local authentication.

---

## Factory Reset Considerations

A factory reset may remove the working local configuration and require provisioning again.

Do not factory-reset the current working H100 solely to test reproducibility unless rebuilding the configuration is acceptable.

Before any factory reset, record:

    H100 hardware revision
    H100 firmware version
    Home Assistant version
    TP-Link integration version
    Current H100 IP configuration
    Current firewall configuration
    Current entity list
    Current automation configuration

This information will be important if the provisioning behavior changes.

---

## Why This Architecture Is Useful

The implementation treats the doorbell as a collection of independent functions:

    Physical Input
         +
    Bluetooth Transport
         +
    Automation Controller
         +
    Audible Output
         +
    Optional Independent Video

For this deployment:

    Physical Input:
        GoveeLife Wireless Mini Smart Button Sensor

    Bluetooth Transport:
        ESP32 ESPHome Bluetooth proxies

    Home Assistant Integration:
        Govee Bluetooth

    Automation Controller:
        Home Assistant OS

    Audible Output:
        TP-Link Tapo H100

    Video:
        Independent integration if required

This approach avoids requiring one vendor to provide every component.

---

## Component Replacement

Because the functions are separated, individual components can be replaced independently.

### Button Replacement

A future button only needs to provide a usable Home Assistant event.

    New Button
        |
        v
    Home Assistant

The H100 automation can remain largely unchanged.

### Bluetooth Transport Replacement

The physical button transport layer can also be modified independently if Home Assistant supports an appropriate reception method.

The automation does not need to understand which Bluetooth receiver observed the device.

### Chime Replacement

A future chime only needs to expose a Home Assistant-controllable audible output.

    Home Assistant
          |
          v
      New Chime

The GoveeLife button can remain unchanged.

### Automation Changes

The physical hardware does not need to change when automation logic changes.

Home Assistant can modify:

- Chime duration.
- Chime tone.
- Volume.
- Time-of-day behavior.
- Notification behavior.
- Lighting behavior.
- Presence conditions.
- Security actions.

---

## Cost Efficiency

The implementation provides a functional Home Assistant doorbell path for approximately:

    $33.09

when allocating one of the two GoveeLife buttons to the doorbell.

The total newly purchased hardware was:

    $43.19

and provides an additional wireless button for another automation.

Existing Home Assistant and ESP32 Bluetooth proxy infrastructure is not included in this incremental cost.

The system therefore avoids paying for integrated functionality that is not required, particularly a dedicated doorbell camera.

---

## Design Principle

The central design principle is:

> A doorbell does not need to be a single proprietary device.

The required functions can be decomposed:

    Visitor Input
         |
         v
    Bluetooth Transport
         |
         v
    Home Assistant Event
         |
         v
    Automation Logic
         |
         +------> Audible Chime
         |
         +------> Optional Notification
         |
         +------> Optional Lighting
         |
         +------> Optional Video Reference
         |
         +------> Future Actions

Home Assistant becomes the integration layer rather than requiring the physical doorbell hardware to provide every feature itself.

---

## Known Limitations

The current deployment has the following known limitations:

- Initial H100 provisioning required temporary installation of the Tapo application.
- Initial provisioning required creation of a temporary TP-Link ID.
- The exact H100 local authentication mechanism has not been independently analyzed.
- Permanent operation after TP-Link ID deletion remains pending verification.
- Future H100 firmware behavior is unknown.
- The incomplete-cloud-onboarding behavior has currently been observed on one deployment.
- Reproducibility on another factory-reset H100 has not been verified.
- Future Tapo application changes could alter the provisioning sequence.
- Future Home Assistant TP-Link integration changes could alter authentication behavior.
- Future Govee Bluetooth or ESPHome changes could alter the Bluetooth input path.
- Individual redundant Bluetooth reception through both deployed ESP32 proxies has not been independently validated.
- Weatherproof and water-resistant operation of the GoveeLife H5122 / B5122 has not been tested as part of this implementation.

### Environmental Placement

The GoveeLife H5122 / B5122 has not been tested in this deployment for weatherproof or water-resistant operation.

No weatherproof capability is assumed as part of this implementation.

The installed button is located in a protected area where direct exposure to most weather conditions is not expected to be an operational concern.

The installation location provides protection from normal direct exposure such as:

- Rain.
- Snow.
- Standing water.
- Direct precipitation.

Environmental exposure has not been formally tested, and this deployment does not establish an ingress-protection, waterproof, or weatherproof rating for the device.

The H5122 / B5122 should therefore not be documented as weatherproof or outdoor-rated based on this implementation.

---

## Final Validation Checklist

Complete this checklist after TP-Link confirms permanent deletion of the TP-Link ID.

    [ ] TP-Link confirms TP-Link ID deletion

    [ ] Tapo application removed from phone

    [ ] H100 remains blocked from WAN

    [ ] ESP32 Bluetooth proxy infrastructure available

    [ ] Govee Bluetooth integration available

    [ ] H5122 doorbell device available

    [ ] GoveeLife button press event is received

    [ ] H100 remains available in Home Assistant

    [ ] Direct Home Assistant H100 siren activation works

    [ ] Doorbell automation executes

    [ ] H100 rings for 1 second

    [ ] Home Assistant can change H100 alarm sound

    [ ] H100 power cycled

    [ ] H100 reconnects to IoT Wi-Fi

    [ ] Home Assistant reconnects to H100

    [ ] GoveeLife button rings H100 after H100 power cycle

    [ ] TP-Link Smart Home integration reloaded

    [ ] H100 remains available after integration reload

    [ ] Doorbell works after integration reload

    [ ] Home Assistant OS restarted

    [ ] ESPHome Bluetooth proxy infrastructure returns after
        Home Assistant OS restart

    [ ] GoveeLife button automatically becomes available after
        Home Assistant OS restart

    [ ] H100 automatically becomes available after
        Home Assistant OS restart

    [ ] Doorbell works after Home Assistant OS restart

    [ ] H100 remains functional with no WAN access throughout testing

If every applicable test passes after TP-Link ID deletion, the deployment can be documented as verified for continued local operation without an ongoing TP-Link account or H100 Internet connection under the tested configuration.

---

## Search Keywords

### TP-Link Tapo H100

**TP-Link Tapo H100**, **Tapo H100 Home Assistant**, **Tapo H100 Home Assistant OS**, **Tapo H100 local control**, **Tapo H100 without cloud**, **Tapo H100 without Internet**, **Tapo H100 TP-Link Smart Home**, **Tapo H100 local polling**, **Tapo H100 IoT network**, **Tapo H100 firewall**, **Tapo H100 blocked Internet**, **Tapo H100 failed onboarding**, **Tapo H100 no devices in Tapo app**, **Tapo H100 Home Assistant chime**, **Tapo H100 doorbell chime**, **Tapo H100 siren Home Assistant**, **Tapo H100 one second chime**, **Tapo H100 account deletion**, **TP-Link ID deletion**

### GoveeLife B5122 / H5122

**GoveeLife Wireless Mini Smart Button**, **GoveeLife B5122**, **Govee H5122**, **Govee Bluetooth H5122**, **GoveeLife button Home Assistant**, **GoveeLife button Home Assistant OS**, **GoveeLife doorbell Home Assistant**, **GoveeLife button event Home Assistant**, **GoveeLife Bluetooth button**, **GoveeLife H5122 weatherproof**, **GoveeLife B5122 weather resistance**, **GoveeLife button protected installation**

### ESPHome Bluetooth Proxy

**ESPHome Bluetooth Proxy**, **ESP32 Bluetooth Proxy**, **Home Assistant Bluetooth Proxy**, **Govee ESPHome Bluetooth Proxy**, **ESP32 Govee Bluetooth**, **distributed Bluetooth Home Assistant**

### Home Assistant Doorbell

**Home Assistant wireless doorbell button**, **Home Assistant custom doorbell**, **Home Assistant local doorbell**, **Home Assistant cross-vendor doorbell**, **Home Assistant doorbell automation**, **Home Assistant siren doorbell**, **Home Assistant inexpensive doorbell**, **Home Assistant doorbell without camera**, **local smart doorbell without cloud**

---

## Revision History

| Revision | Date | Description | Author |
| --- | --- | --- | --- |
| **1.0.0** | 2026-08-27 | Initial documentation of Tapo H100 incomplete cloud onboarding, local Home Assistant integration, WAN-isolated operation, power-cycle validation, and pending TP-Link ID deletion testing. | projectfong |
