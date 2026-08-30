# Google Nest Device Access with Home Assistant OS

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

This document records the implementation of Google Nest Device Access with Home Assistant OS (HAOS), including the Google Cloud project, Smart Device Management API, OAuth configuration, Nest Device Access registration, Google Cloud Pub/Sub event delivery, segmented-network browser callback requirements, and final discovery of Nest cameras and a thermostat in Home Assistant.

The completed implementation allows Home Assistant to act as the primary user-facing control and automation layer for the authorized Nest devices while Google continues to provide the required Nest cloud backend.

The validated deployment integrated:

- Two Google Nest outdoor cameras.
- One Google Nest thermostat.
- Camera livestream access.
- Camera event access.
- Camera snapshot access for supported events.
- Thermostat status and control.
- Google Cloud Pub/Sub event delivery.
- Three Home Assistant devices.
- Eight Home Assistant entities.

The Home Assistant Google Nest integration requires Internet access because communication with Nest devices occurs through Google's Smart Device Management API.

Estimated implementation time:

```text
30-60 minutes
```

Additional time may be required when local DNS, OAuth, browser routing, or segmented-network policy requires troubleshooting.

---

## Quick Start

This section provides the clean implementation path without the troubleshooting detours encountered during the original deployment.

### 1. Start the Nest Integration in Home Assistant

In Home Assistant:

```text
Settings
-> Devices & services
-> Add Integration
-> Nest
```

Follow the Home Assistant setup wizard.

### 2. Create a Dedicated Google Cloud Project

From the first Home Assistant Nest setup screen, open the provided Google Cloud Console link.

Create a dedicated project.

Example:

```text
Project name:
HAOS NEST
```

Record the generated Google Cloud Project ID.

Example format:

```text
haos-nest-123456
```

Do not confuse the following identifiers:

```text
Google Cloud Project ID
```

and:

```text
Nest Device Access Project ID
```

They are different identifiers created by different Google services.

### 3. Enable the Required Google APIs

Return to the Home Assistant Nest setup dialog.

Use the API links directly from the Home Assistant dialog.

Enable:

```text
Smart Device Management API
Cloud Pub/Sub API
```

Do not use unrelated Google Cloud Hub buttons such as:

```text
Enable Required APIs
Enable Maintenance API
Enable App Optimize API
Enable Gemini Cloud Assist API
Set up in App Hub
```

Those Google Cloud Hub services are not required for the Nest integration.

### 4. Configure Google OAuth

When Home Assistant requests application credentials, open the provided OAuth configuration link.

Configure the Google Auth Platform.

Use:

```text
App name:
HAOS NEST

Audience:
External

Publishing status:
Testing
```

Set the required support and developer contact email addresses to the Google account used to manage the project.

Do not expose those email addresses in public documentation or screenshots.

### 5. Add the Google Account as a Test User

Because the OAuth application is operating in Testing mode, add the Google account used with the Nest home as a test user.

Navigate to:

```text
Google Auth Platform
-> Audience
-> Test users
```

Add the same Google account used for Nest access.

Failure to add the account as a test user may result in:

```text
Access blocked
```

or:

```text
Google has not verified this app
```

The second message is expected for a personal OAuth application operating in Testing mode after the account has been added as a test user.

### 6. Create the OAuth Client

Create an OAuth client.

Use:

```text
Application type:
Web application

Name:
HAOS NEST
```

Leave:

```text
Authorized JavaScript origins:
Empty
```

Add exactly this Authorized Redirect URI:

```text
https://my.home-assistant.io/redirect/oauth
```

Create the OAuth client.

Securely record:

```text
OAuth Client ID
OAuth Client Secret
```

Do not place the OAuth Client Secret in:

```text
Git
Gitea
GitHub
Wiki.js
Public documentation
Screenshots
Issue trackers
Chat logs
```

If the OAuth credential JSON file is downloaded as a backup, treat the JSON file as a secret.

### 7. Add the OAuth Credentials to Home Assistant

Return to Home Assistant.

Enter:

```text
Name:
HAOS NEST

OAuth client ID:
<REDACTED>

OAuth client secret:
<REDACTED>
```

Select:

```text
Add
```

### 8. Enter the Google Cloud Project ID

When Home Assistant requests:

```text
Google Cloud Project ID
```

enter the Google Cloud Project ID created earlier.

Example:

```text
haos-nest-123456
```

Do not enter:

```text
HAOS NEST
```

because that is the display name, not the Project ID.

### 9. Register for Google Nest Device Access

Home Assistant will provide a link to the Google Device Access Console.

Open the Device Access Console and complete the required one-time registration payment.

At the time of this implementation, the registration fee was:

```text
US $5 one-time fee
```

After registration, select:

```text
Create project
```

Use:

```text
Project name:
HAOS NEST
```

### 10. Add the OAuth Client ID to Device Access

When prompted for the OAuth Client ID, enter the OAuth Client ID created in Google Cloud.

Enter:

```text
OAuth Client ID:
<REDACTED>
```

Do not enter the OAuth Client Secret.

### 11. Leave Events Disabled During Initial Device Access Creation

During Device Access project creation, Google will ask whether events should be enabled.

Leave:

```text
Enable events:
Unchecked
```

Then select:

```text
Create project
```

Home Assistant will generate the correct Pub/Sub topic later in the setup process.

### 12. Enter the Device Access Project ID into Home Assistant

After the Device Access project is created, record the generated Device Access Project ID.

Return to Home Assistant.

Enter the value into:

```text
Device Access Project ID
```

Do not enter the Google Cloud Project ID in this field.

### 13. Authorize the Nest Home and Devices

Google will display the Nest permissions page.

Enable:

```text
Allow HAOS NEST to see information about your home
```

Then select the Nest devices and capabilities that Home Assistant should access.

The validated implementation authorized:

```text
Camera 1:
Livestream enabled
Camera events enabled
Camera snapshots enabled

Camera 2:
Livestream enabled
Camera events enabled
Camera snapshots enabled

Thermostat:
Access and control enabled
```

Only authorize devices and capabilities that should be exposed to Home Assistant.

### 14. Authorize the Required Google OAuth Permissions

Google will display the OAuth permissions requested by the Home Assistant Nest integration.

Enable:

```text
See and/or control the devices that you selected
```

and:

```text
View and manage Pub/Sub topics and subscriptions
```

Both were required by the validated implementation.

### 15. Configure a Browser-Reachable Home Assistant URL

The OAuth flow uses:

```text
https://my.home-assistant.io/redirect/oauth
```

to redirect the browser back to the local Home Assistant instance.

The browser performing the OAuth authorization must be able to resolve and reach the configured HAOS URL.

Recommended:

```text
http://<local-haos-fqdn>:8123
```

Example:

```text
http://haos.example.internal:8123
```

A direct IP address also works:

```text
http://<haos-ip-address>:8123
```

Example:

```text
http://192.168.x.x:8123
```

Do not assume that:

```text
http://homeassistant.local:8123
```

will resolve across segmented networks.

The browser must be able to reach HAOS.

Google does not require inbound access to the local Home Assistant hostname.

### 16. Link the Account to Home Assistant

After configuring the My Home Assistant instance URL, select:

```text
Link account
```

The browser should redirect back to the local HAOS instance.

If the local hostname cannot be resolved or routed from the browser network, the redirect will fail.

### 17. Let Home Assistant Create the Pub/Sub Topic

Home Assistant will display:

```text
Configure Cloud Pub/Sub topic
```

Leave:

```text
Create new topic
```

selected.

Select:

```text
Submit
```

Home Assistant creates the required Pub/Sub topic.

### 18. Enable Events in the Device Access Console

Home Assistant will provide a generated Pub/Sub topic path.

Example format:

```text
projects/<google-cloud-project-id>/topics/home-assistant-<generated-value>
```

Copy the complete topic path.

Return to:

```text
Device Access Console
-> HAOS NEST
-> Events
```

Enable the Pub/Sub topic for events.

Paste the complete topic path.

Select:

```text
Add & validate
```

Google publishes a validation event to confirm:

- The topic exists.
- The topic path is valid.
- Device Access has permission to publish events.

### 19. Let Home Assistant Create the Pub/Sub Subscription

Return to Home Assistant.

Home Assistant will display:

```text
Configure Cloud Pub/Sub subscription
```

Leave:

```text
Create new subscription
```

selected.

Select:

```text
Submit
```

Home Assistant creates the subscription required to receive Nest device updates.

### 20. Assign Devices and Areas

After authentication completes, Home Assistant should display:

```text
Successfully authenticated
```

The authorized Nest devices should appear.

Assign the devices to the appropriate Home Assistant areas.

Example:

```text
Camera
-> Backyard

Camera
-> Outdoor Patio

Thermostat
-> Hallway
```

Select:

```text
Finish
```

### 21. Validate the Integration

Open:

```text
Settings
-> Devices & services
-> Google Nest
```

The validated deployment reported:

```text
3 devices
8 entities
```

Validate:

- Both cameras appear.
- Both cameras expose expected entities.
- Camera livestreams load.
- Camera events are available where supported.
- Camera snapshots are available for supported events.
- The thermostat appears.
- Thermostat state is available.
- Thermostat temperature is available.
- Thermostat setpoint control works.
- Home Assistant automations can reference the Nest entities.

---

## Final Validated State

The completed Home Assistant Google Nest integration reported:

```text
Google Nest
Requires Internet

3 devices
8 entities
```

Device summary:

| Device | Type | Home Assistant Entities |
| --- | --- | ---: |
| Backyard | Nest Camera | 2 |
| Backyard Patio | Nest Camera | 2 |
| Hallway | Nest Thermostat | 4 |

Validation Result:

```text
Google Nest authentication completed successfully.
Both Nest cameras were discovered.
The Nest thermostat was discovered.
Google Cloud Pub/Sub was configured.
Home Assistant created the required Pub/Sub subscription.
Three Nest devices and eight entities were available in HAOS.
```

---

## Architecture

The final architecture is:

```text
+------------------------------------------------------------+
|                    Google Nest Devices                     |
|                                                            |
|   +----------------+  +----------------+  +--------------+ |
|   | Outdoor Camera |  | Outdoor Camera |  |  Thermostat  | |
|   +--------+-------+  +--------+-------+  +------+-------+ |
|            |                   |                 |         |
+------------+-------------------+-----------------+---------+
             |                   |                 |
             +-------------------+-----------------+
                                 |
                                 v
                     +-----------------------+
                     |   Google Nest Cloud   |
                     +-----------+-----------+
                                 |
                                 v
               +----------------------------------+
               | Smart Device Management API      |
               +----------------+-----------------+
                                |
                 +--------------+--------------+
                 |                             |
                 v                             v
      +----------------------+      +----------------------+
      | Device State/Control |      | Nest Device Events   |
      +----------+-----------+      +----------+-----------+
                 |                             |
                 |                             v
                 |                  +----------------------+
                 |                  | Google Cloud Pub/Sub |
                 |                  | Topic                |
                 |                  +----------+-----------+
                 |                             |
                 |                             v
                 |                  +----------------------+
                 |                  | Pub/Sub Subscription |
                 |                  +----------+-----------+
                 |                             |
                 +-------------+---------------+
                               |
                               v
                    +-------------------------+
                    | Home Assistant OS       |
                    | Google Nest Integration |
                    +------------+------------+
                                 |
                 +---------------+---------------+
                 |               |               |
                 v               v               v
             Dashboards      Automations     Notifications
```

---

## Control Plane vs. Backend

Home Assistant becomes the primary control and automation interface.

Google remains part of the backend.

```text
User
 |
 v
Home Assistant OS
 |
 +-> Dashboards
 |
 +-> Automations
 |
 +-> Notifications
 |
 +-> Nest Integration
       |
       v
Google Smart Device Management API
       |
       v
Nest Devices
```

This means routine interaction can occur through Home Assistant without opening the Google Home or Nest application.

However, this does not convert Nest devices into locally controlled devices.

The Google Nest integration remains dependent on:

```text
Internet connectivity
Google OAuth
Google Device Access
Smart Device Management API
Google Cloud Pub/Sub
Google Nest cloud services
```

---

## Why Home Assistant Becomes the Primary Interface

Once the Nest devices are integrated, Home Assistant can combine them with devices from other ecosystems.

Example:

```text
Nest Camera Event
      |
      v
Home Assistant Automation
      |
      +-> Trigger local light
      |
      +-> Send notification
      |
      +-> Trigger Tapo device
      |
      +-> Trigger Aqara device
      |
      +-> Record automation state
```

Google does not need awareness of the non-Google devices participating in the Home Assistant automation.

This allows Home Assistant to serve as the unified automation layer across otherwise separate ecosystems.

---

## Google Cloud Project

A dedicated Google Cloud project was created for the Home Assistant Nest integration.

Example:

```text
Project name:
HAOS NEST
```

Using a dedicated project keeps the integration isolated from unrelated Google Cloud workloads.

### Required APIs

Only the required APIs were enabled:

```text
Smart Device Management API
Cloud Pub/Sub API
```

### APIs Not Required

The Google Cloud interface may suggest additional APIs.

Examples encountered during implementation included:

```text
Maintenance API
App Optimize API
Cloud Hub required APIs
Gemini Cloud Assist API
```

These were not required for the Home Assistant Nest integration.

Validation Result:

```text
Smart Device Management API: Enabled
Cloud Pub/Sub API: Enabled
Unrelated Cloud Hub APIs: Not required
```

---

## Google OAuth Configuration

Google OAuth provides the authentication and authorization path between Home Assistant and Google Nest.

The validated OAuth configuration used:

```text
Application type:
Web application
```

OAuth client name:

```text
HAOS NEST
```

Authorized JavaScript origins:

```text
None
```

Authorized Redirect URI:

```text
https://my.home-assistant.io/redirect/oauth
```

### OAuth Audience

The OAuth application was configured as:

```text
External
```

with:

```text
Publishing status:
Testing
```

The Google account used for Nest was explicitly added as a test user.

### Why External Was Used

The Google account was not being restricted to users inside a Google Workspace organization.

External allows the designated Google account to authenticate while the application remains in Testing mode.

External does not mean the Home Assistant instance itself is publicly exposed.

---

## OAuth Credential Security

The OAuth client secret is sensitive.

The following values must not be published:

```text
OAuth Client Secret
OAuth credential JSON
Access tokens
Refresh tokens
Authorization codes
Google account identifiers when privacy is required
```

The OAuth Client ID is less sensitive than the Client Secret but should still be treated as configuration information rather than unnecessarily exposed data.

### Backup

Google may provide an option to download the OAuth client configuration as JSON.

If downloaded, secure the JSON file appropriately.

Do not commit it to Git.

Example ignore concept:

```text
client_secret*.json
oauth*.json
google_credentials*.json
```

The exact local filename may vary.

---

## Nest Device Access Project

Google Nest Device Access is separate from the Google Cloud project.

The Device Access Console was used to create:

```text
HAOS NEST
```

The project was created in:

```text
Sandbox
```

mode.

Commercial development approval was not required for this personal Home Assistant deployment.

### Device Access User Limit

The created Sandbox project displayed:

```text
25 users across 5 structures
```

This was sufficient for the personal Home Assistant deployment.

### Device Access Scope

The Device Access project displayed the required scope:

```text
https://www.googleapis.com/auth/sdm.service
```

---

## Device Authorization

Google allows granular authorization of Nest devices and capabilities.

The validated deployment enabled home information:

```text
Home information:
Enabled
```

The two cameras were authorized for:

```text
Livestream
Camera events
Camera snapshots
```

The thermostat was authorized for:

```text
Access and control
```

This authorization model allows Home Assistant access to only the devices and capabilities intentionally selected.

---

## Camera Integration

Two existing Nest outdoor cameras were integrated.

The cameras had already been operating for many years before the Home Assistant integration.

No camera replacement was required.

The integration added Home Assistant access while retaining the existing Nest backend.

### Camera Capabilities Authorized

For each camera:

```text
Livestream:
Enabled

Camera events:
Enabled

Camera snapshots:
Enabled
```

The exact entities and event capabilities exposed in Home Assistant depend on the Nest model and Smart Device Management API support.

### Camera Validation

Validate each camera by opening:

```text
Settings
-> Devices & services
-> Google Nest
-> <Camera>
```

Confirm:

```text
Camera entity exists
Camera state is available
Livestream loads
Supported events are received
Supported snapshots are accessible
```

---

## Thermostat Integration

The Nest thermostat was authorized for:

```text
Access and control
```

The completed integration exposed four Home Assistant entities for the thermostat.

Validate:

```text
Current temperature
Thermostat state
Configured setpoint
Climate control
```

The exact entities depend on thermostat model and supported Google SDM traits.

---

## Google Cloud Pub/Sub

Google Cloud Pub/Sub provides asynchronous Nest event delivery to Home Assistant.

The event path is:

```text
Nest Device
    |
    v
Google Nest
    |
    v
Smart Device Management API
    |
    v
Pub/Sub Topic
    |
    v
Pub/Sub Subscription
    |
    v
Home Assistant OS
```

### Topic Creation

Home Assistant was allowed to create a new Pub/Sub topic.

The generated topic uses a format similar to:

```text
projects/<project-id>/topics/home-assistant-<generated-value>
```

The generated suffix should not be manually invented.

Copy the exact value displayed by Home Assistant.

### Device Access Event Configuration

After Home Assistant created the topic, the topic was entered into the Device Access Console.

Google Device Access then performed:

```text
Add & validate
```

This validated that:

- The topic existed.
- The topic path was valid.
- Device Access could publish events.

### Subscription Creation

Home Assistant created a new Pub/Sub subscription automatically.

This subscription allows Home Assistant to receive the Nest events published to the topic.

---

## Segmented Network Considerations

The home network remains segmented.

The Google Nest integration does not require flattening VLAN or firewall boundaries.

The OAuth browser flow does create one important requirement:

```text
The browser performing OAuth must be able to reach the configured Home Assistant URL.
```

### Failed Initial Callback

The initial callback used:

```text
http://homeassistant.local:8123
```

The browser was located on a segmented network where that `.local` hostname was not reachable from the browser.

The browser displayed an error similar to:

```text
We cannot connect to the server at homeassistant.local.
```

### Cause

`.local` normally relies on multicast DNS.

Even when mDNS is available elsewhere in the network, the browser's VLAN or routing policy may prevent that hostname from resolving or reaching HAOS.

### Fix

Use a deterministic local DNS record or an IP address that the browser can resolve and reach.

Recommended:

```text
http://<local-haos-fqdn>:8123
```

Alternative:

```text
http://<haos-ip>:8123
```

No network flattening is required.

### Preferred Design

For segmented networks, normal local DNS is generally preferable to depending exclusively on `.local` mDNS naming.

Example:

```text
haos.example.internal
        |
        v
Local DNS
        |
        v
HAOS IP
```

Firewall policy can then explicitly control whether the management client is permitted to access:

```text
HAOS TCP/8123
```

---

## My Home Assistant Redirect Behavior

The OAuth redirect path is:

```text
Google OAuth
     |
     v
https://my.home-assistant.io/redirect/oauth
     |
     v
Browser
     |
     v
Configured local HAOS URL
     |
     v
Home Assistant OS
```

The local HAOS URL is used by the browser.

Google does not need direct network access to:

```text
homeassistant.local
```

or:

```text
<local-haos-fqdn>
```

This distinction is important in segmented networks.

---

## Troubleshooting

## Issue: Google Cloud Interface Shows "Enable Required APIs"

### Cause

Google Cloud Hub displays generic recommendations for Cloud Hub features.

These are unrelated to the Home Assistant Nest integration.

### Fix

Do not use the Cloud Hub:

```text
Enable Required APIs
```

button.

Use the direct links supplied by the Home Assistant Nest integration and enable only:

```text
Smart Device Management API
Cloud Pub/Sub API
```

### Validation Result

```text
Only the required Nest integration APIs are enabled.
```

---

## Issue: OAuth Access Blocked

Example:

```text
Access blocked
```

### Cause

The OAuth application is in Testing mode and the Google account attempting authentication has not been added as a test user.

### Fix

Navigate to:

```text
Google Auth Platform
-> Audience
-> Test users
```

Add the Google account used for Nest.

Retry the OAuth flow.

### Expected Follow-Up

Google may display:

```text
Google has not verified this app
```

This is expected for the personal test application.

If the account was intentionally added as a test user and the application was created by the administrator performing the setup, select:

```text
Continue
```

### Validation Result

```text
OAuth authorization proceeds to the requested permissions.
```

---

## Issue: Browser Cannot Reach homeassistant.local

### Cause

The browser is on a segmented network and cannot resolve or reach the HAOS `.local` address.

### Fix

Configure My Home Assistant with a browser-reachable local FQDN:

```text
http://<local-haos-fqdn>:8123
```

or use the HAOS IP address:

```text
http://<haos-ip>:8123
```

Verify from the same browser before repeating OAuth.

### Validation

Open:

```text
http://<local-haos-fqdn>:8123
```

Expected result:

```text
Home Assistant loads successfully.
```

---

## Issue: OAuth Redirect Displays 400 Bad Request

### Cause

A redirect error can occur during the OAuth handoff when the configured Home Assistant destination or browser state does not match the active OAuth flow.

During the validated implementation, a 400 response appeared during a retry, but Home Assistant had successfully advanced to Pub/Sub configuration.

### Fix

Before resetting or deleting the integration, return to Home Assistant and inspect the active Nest setup dialog.

If Home Assistant has advanced to:

```text
Configure Cloud Pub/Sub topic
```

continue the setup.

Do not recreate the entire Google Cloud or Device Access configuration unless Home Assistant actually reports an unrecoverable authentication failure.

### Validation Result

```text
Home Assistant continued into Pub/Sub configuration.
OAuth had completed sufficiently for the integration to proceed.
```

---

## Issue: Device Access Events Are Disabled

### Cause

Events were intentionally left disabled during initial Device Access project creation.

This is expected.

Home Assistant must first create the Pub/Sub topic.

### Fix

Allow Home Assistant to create the topic.

Copy the complete generated topic path.

Return to Device Access.

Enable events and enter the generated topic.

Select:

```text
Add & validate
```

### Validation Result

```text
Device Access validates the Pub/Sub topic successfully.
```

---

## Issue: Pub/Sub Topic vs. Subscription Confusion

A Pub/Sub topic and subscription are different objects.

The topic receives published Nest events:

```text
Nest
-> Pub/Sub Topic
```

The subscription allows Home Assistant to consume those events:

```text
Pub/Sub Topic
-> Subscription
-> Home Assistant
```

Home Assistant can create both during the setup process.

---

## Issue: Google Cloud Project ID vs. Device Access Project ID

These identifiers are not interchangeable.

### Google Cloud Project ID

Example format:

```text
haos-nest-123456
```

Used for:

```text
Google Cloud APIs
Pub/Sub
OAuth project association
```

### Device Access Project ID

Generated separately by the Google Nest Device Access Console.

Used by Home Assistant to identify the Nest Device Access project.

### Fix

Always copy the value directly from the requested Google interface instead of assuming the IDs are the same.

---

## Security and Isolation Notes

### Google Does Not Manage Home Assistant

The Nest integration grants Home Assistant authorization to access selected Google/Nest resources.

It does not grant Google administrative control over the HAOS instance.

Google does not receive authority to manage:

```text
HAOS users
HAOS dashboards
HAOS automations
HAOS integrations unrelated to Google
Local Zigbee devices
Local Matter devices
Local Tapo devices
Local Aqara devices
VLAN configuration
Firewall policy
Home Assistant host administration
```

### Google Remains a Cloud Dependency

Google remains responsible for:

```text
Google Account authentication
OAuth authorization
Nest Device Access
Smart Device Management API
Google Cloud Pub/Sub
Nest cloud connectivity
```

If those cloud services are unavailable, corresponding Nest functionality in Home Assistant may be unavailable.

### Credentials

Protect:

```text
OAuth Client Secret
OAuth credential JSON
Access tokens
Refresh tokens
Authorization codes
```

Do not expose these values in screenshots or public documentation.

### Least Privilege

Authorize only the Nest devices and capabilities required by the deployment.

The validated deployment intentionally authorized:

```text
2 cameras
1 thermostat
```

with the specific camera and thermostat capabilities required for Home Assistant operation.

### Network Segmentation

Network segmentation remains in place.

Only required communication should be permitted.

Do not flatten VLAN boundaries to resolve OAuth callback problems.

Use:

```text
Local DNS
Explicit routing
Explicit firewall policy
```

instead.

---

## Operational Behavior

Home Assistant can now be used as the primary control interface for the integrated Nest devices.

Typical routine interaction can occur through:

```text
Home Assistant dashboards
Home Assistant automations
Home Assistant notifications
Home Assistant scripts
Home Assistant scenes
```

The Google Home or Nest application is no longer required for most routine Home Assistant-based interaction with these authorized devices.

However, the Google application may still be required for:

```text
Nest account administration
Google account administration
Subscription management
Certain device-specific settings
Device recovery
Google-specific troubleshooting
```

---

## Automation Opportunities

Once Nest devices are exposed to Home Assistant, they can participate in automations involving non-Google devices.

Examples include:

```text
Nest camera person event
-> Home Assistant
-> Turn on exterior light
```

```text
Nest camera motion event
-> Home Assistant
-> Send mobile notification
```

```text
Nest camera event
-> Home Assistant
-> Trigger local audible notification
```

```text
Nest thermostat state
-> Home Assistant
-> Adjust other HVAC-related automation
```

```text
Nest camera event
-> Home Assistant
-> Trigger Tapo, Matter, Zigbee, or other local device
```

This is the primary architectural benefit of integrating Nest into Home Assistant.

---

## Validation Checklist

Use the following checklist after completing the integration.

### Google Cloud

```text
[ ] Dedicated Google Cloud project exists
[ ] Smart Device Management API enabled
[ ] Cloud Pub/Sub API enabled
[ ] OAuth client configured as Web application
[ ] Authorized redirect URI configured correctly
[ ] OAuth Client Secret stored securely
```

### Google Auth Platform

```text
[ ] Audience configured appropriately
[ ] OAuth application configured
[ ] Google Nest account added as test user when using Testing mode
[ ] OAuth authorization completes
```

### Device Access

```text
[ ] Device Access registration completed
[ ] Device Access project created
[ ] OAuth Client ID associated
[ ] Required Nest devices authorized
[ ] Camera livestream access authorized
[ ] Camera event access authorized
[ ] Camera snapshot access authorized
[ ] Thermostat control authorized
[ ] Pub/Sub events enabled after topic creation
[ ] Pub/Sub topic validated
```

### Home Assistant

```text
[ ] Google Cloud Project ID accepted
[ ] Device Access Project ID accepted
[ ] OAuth authentication completed
[ ] Pub/Sub topic created
[ ] Pub/Sub subscription created
[ ] Cameras discovered
[ ] Thermostat discovered
[ ] Areas assigned
[ ] Integration setup completed
```

### Network

```text
[ ] Browser can resolve local HAOS FQDN
[ ] Browser can reach HAOS TCP/8123 or configured HTTPS endpoint
[ ] Segmentation remains intact
[ ] No unnecessary broad firewall rule added
[ ] Google does not require inbound access to local HAOS
```

### Functional Validation

```text
[ ] Camera 1 available in HAOS
[ ] Camera 2 available in HAOS
[ ] Camera livestream tested
[ ] Camera event entities reviewed
[ ] Camera snapshot capability reviewed
[ ] Thermostat available in HAOS
[ ] Thermostat state tested
[ ] Thermostat control tested
[ ] Google Nest integration reports expected devices and entities
```

---

## Validated Deployment Result

The final deployment reported:

```text
Google Nest
Requires Internet

3 devices
8 entities
```

Device distribution:

```text
Backyard Camera:
2 entities

Backyard Patio Camera:
2 entities

Hallway Thermostat:
4 entities
```

The final architecture provides Home Assistant with unified access to Nest devices while retaining the segmented network and the existing Google Nest backend.

---

## Related Search Keywords

**Home Assistant Nest**  
**Home Assistant OS Google Nest**  
**HAOS Nest integration**  
**Google Nest Device Access**  
**Google Smart Device Management API**  
**SDM API Home Assistant**  
**Google Nest OAuth Home Assistant**  
**Google Cloud Pub/Sub Home Assistant**  
**Nest camera Home Assistant**  
**Nest thermostat Home Assistant**  
**Nest camera livestream HAOS**  
**Nest camera events HAOS**  
**Nest camera snapshots Home Assistant**  
**Google OAuth test user Home Assistant**  
**home-assistant.io OAuth redirect**  
**My Home Assistant redirect**  
**Home Assistant local FQDN**  
**Home Assistant segmented network**  
**Nest Device Access Pub/Sub topic**  
**Nest Pub/Sub subscription**  

---

## Revision Control

| Version | Date | Summary | Author |
| -------- | ---- | ------- | ------ |
| **1.0.0** | 2026-08-29 | Initial documentation of Google Nest Device Access, OAuth, Smart Device Management API, Pub/Sub, segmented-network callback handling, two Nest cameras, and Nest thermostat integration with Home Assistant OS. | projectfong |
