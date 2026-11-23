# ALUSSO Galaxy Link LAN groups

This optional transport setting sends commands to an existing Galaxy Link group
through its Wi-Fi member. It was developed from a Tuya Smart packet capture and
a standalone script that the owner verified switched all four lights off/on
without internet or phone Bluetooth. The patched Homebridge runtime still needs
the same physical verification. This fork branch identifies itself as
`4.4.0-alusso.1`; it is not an upstream npm release.

## Install the fork build

Back up Homebridge through its UI and record the currently installed Tuya Plus
version. Stop the Tuya Plus child bridge. In the same environment and installation
location as the existing plugin, install this branch:

```sh
npm install -g 'github:Makk1000/homebridge-tuya-plus#alusso-local-groups'
```

That command applies to a global npm installation. For a local/prefix-based
installation, use that existing prefix instead of `-g`. If Homebridge runs in
Docker, run npm inside its container, not on the Mac host. Confirm the installation
location before changing it. The package name, platform name and UUID generation
remain unchanged. Add the device configuration below, then restart only the
Tuya Plus child bridge. A future registry plugin update can replace this fork
build, so retain the branch URL and check group support before updating.

## Configuration

Add the following fields to the **existing Wi-Fi member's device entry**:

```json
{
  "forceVersion": "3.3",
  "lanGroup": {
    "cid": "0058",
    "mbid": "c020"
  }
}
```

Keep its existing `id`, `key`, IP and accessory type. Do not add another device
entry with that same ID. Keeping the ID preserves the existing accessory UUID.
The addresses above are from this owner's group; they are not universal defaults.
Deleting/recreating a Tuya group may change them.

If no downlight entry exists yet, this is a minimum on/off example. Replace the
device ID, key and IP with the Wi-Fi member's details:

```json
{
  "type": "SimpleLight",
  "name": "Living Room Downlights",
  "id": "YOUR_WIFI_MEMBER_DEVICE_ID",
  "key": "REPLACE_WITH_DOWNLIGHT_1_LOCAL_KEY",
  "ip": "192.168.1.50",
  "forceVersion": "3.3",
  "dpPower": 1,
  "lanGroup": {
    "cid": "0058",
    "mbid": "c020"
  }
}
```

Put that object inside the existing TuyaLan platform's `devices` array. Do not
replace the entire Homebridge config with it. Back up Homebridge through its UI
before installing or editing configuration. Restart only the Tuya Plus child
bridge after applying the patch and configuration.

## Scope and status

- Group power on/off is the first hardware test. Other numeric DPs use the same
  group envelope but brightness, colour and temperature are not yet verified
  through the patched plugin.
- Writes use command 13 with exactly `cid`, `ctype: 2`, `dps` and `mbid`.
- Only Tuya 3.3 is supported for `lanGroup`. Other devices retain their existing
  protocol behavior.
- Group reports must match all three addressing fields. Unaddressed member
  reports cannot overwrite group state after the initial startup query.
- The first normal device-query response seeds characteristic setup. That
  initial reading is the Wi-Fi member's state, not an independent confirmation
  that all four fittings share that state. Group reports update it afterwards.
- Repeated commands are sent even when they match cached group state, so an ON
  request can still bring back a member that was individually switched off.
- Cloud fallback is disabled for this group only. Sending a cloud command to
  the Wi-Fi member ID would not reproduce the LAN group operation.
- This creates one HomeKit light controlling the group. Individual Bluetooth
  fitting control is not implemented.
- An acknowledgement or a group report is not independent feedback from every
  Bluetooth fitting. Check the physical lights during the first test.
- The patch does not change IDs, cached accessories, pairings or other devices'
  configuration. It does not embed any local key.

## First HomeKit test

1. Close Tuya Smart and stop the standalone Python test so those connections are
   released. If LocalTuya is connected to this fitting, temporarily disable only
   that competing connection for the test.
2. Leave the local network powered and connected. Disconnect the router's WAN
   and keep the iPhone's mobile data and Bluetooth off.
3. Turn the HomeKit group accessory OFF, wait a few seconds, then ON. Verify all
   four fittings each time.
4. If it fails, restore internet access and capture the Tuya Plus child-bridge
   log around that operation. Redact keys if sharing config.

## Reverting

Stop the Tuya Plus child bridge. Reinstall the previously recorded registry
version in the same installation location, remove `lanGroup` from the device
entry, and restart that child bridge. Preserve the ID and HomeKit pairing
throughout. The original source is also retained on the fork's `main` branch.
