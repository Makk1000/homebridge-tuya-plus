# ALUSSO Galaxy Link LAN groups

This optional transport setting sends commands to an existing Galaxy Link group
through its Wi-Fi member. It was developed from a Tuya Smart packet capture and
a standalone script that the owner verified switched all four lights off/on
without internet or phone Bluetooth. Group control through this Homebridge
fork has also been verified on all four fittings using the existing light's
`lanGroup` configuration. This branch identifies itself as `4.4.0+alusso.3`;
it is not an upstream npm release.

## Reuse of the existing light controls

Group support is a transport option on the existing light accessory. It does
not define separate power, dimming, colour or white-temperature commands.
`RGBTWLightAccessory` keeps its existing DP mapping, value conversions and
HomeKit handlers. The shared transport takes those same DP values and adds the
group address envelope when `lanGroup` is configured.

For example, the normal white-brightness handler can produce `{"3": 750}`.
The group transport sends that unchanged DP object as:

```json
{
  "cid": "0058",
  "ctype": 2,
  "dps": {"3": 750},
  "mbid": "c020"
}
```

The capture requires Tuya command 13 for this envelope, in place of command 7
used for ordinary Tuya 3.3 device writes. Encryption and packet framing remain
the existing transport implementation. Tests exercise the registered RGB light
handlers for power, white/colour brightness, temperature, hue/saturation and a
custom brightness DP, and verify their DP values are unchanged by group routing.

Brightness and colour already use these existing handlers. Their outstanding
verification is a physical test of all four fittings, not additional command
implementation. Startup state handling, report filtering, repeated group writes
and disabling member-only cloud fallback support that shared transport.

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
Tuya Plus child bridge.

The installed title is **Homebridge Tuya Plus (@Makk1000)**. The package sets
`private: true`, which Homebridge UI supports for skipping npm registry lookups
and update offers. This prevents upstream npm releases from being offered as
updates to the fork; future fork updates must be installed from this branch.
Homebridge UI no longer fills the maintainer label from the upstream npm
listing; the title identifies the fork owner. Original authors remain credited in the package
and README. GitHub repository visibility is unaffected.

## Find group addresses in the setup UI

Open the plugin settings and expand **Find a Tuya LAN group**. The standard
configuration form remains available below the helper. Save a light's device
configuration before using the finder; the device list and local key come from
the saved Homebridge configuration.

1. Select the saved Wi-Fi light.
2. Record a short capture while changing the existing group in Tuya Smart.
   Disable phone Bluetooth so the command uses Wi-Fi. If another local
   controller holds the light's connection, stop that controller for the capture.
3. Choose the `.pcap` or `.pcapng` file and click **Search capture**.
4. Review the observed device address, CID and MBID. If several combinations are
   found, select the group you intended to control.
5. Click **Use selected group** to fill `lanGroup` in the configuration form,
   then review and **Save**. Restart the Tuya Plus child bridge.

A capture must see the phone-to-light TCP connection. Record it on the relevant
access point/router or a mirrored switch port; an ordinary capture on the
Homebridge host may not see traffic between two other devices. For example,
on a capture-capable access point/router, replace the example address with the
Wi-Fi light's address:

```sh
tcpdump -i any -s 0 -w tuya-group.pcap 'host 192.0.2.50 and tcp port 6668'
```

The helper accepts captures up to 16 MiB and searches verified Tuya 3.3 command
13 envelopes with `ctype: 2`, `cid`, `mbid` and DPs. It supports IPv4 TCP
reassembly, Ethernet/VLAN, raw IP and Linux cooked captures. Results are based
on valid frame checksums and decryption with the selected light's saved local
key. No local key, decrypted DPs or raw packets are returned in the result.

This is a capture-assisted finder, not a device group-list query or an automatic
traffic recorder. It sends no commands to lights and does not save configuration
automatically. If no compatible group command is present, it returns no group
rather than guessing an address. Capture analysis runs on the Homebridge server;
the helper does not send the capture to an external service.

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

The `SimpleLight` example exposes switching only. For an existing `RGBTWLight`
entry, retain that type and its DP mappings and add only the group settings.
Its existing brightness, colour and white-temperature controls then use the
same group transport; no separate group light class is required.

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

To roll back from the capture finder to fork version `4.4.0+alusso.2`, stop the
Tuya Plus child bridge and install the preserved build from its exact commit:

```sh
npm install -g --omit=dev 'https://github.com/Makk1000/homebridge-tuya-plus/archive/5e5a55cb46f1ce070f426f58a4368a8358024123.tar.gz'
```

Restart that child bridge. Keep the existing `lanGroup` configuration: this
restores the previous group-control build without the capture finder.

To remove group support entirely:

Stop the Tuya Plus child bridge. Reinstall the previously recorded registry
version in the same installation location, remove `lanGroup` from the device
entry, and restart that child bridge. Preserve the ID and HomeKit pairing
throughout. The original source is also retained on the fork's `main` branch.
