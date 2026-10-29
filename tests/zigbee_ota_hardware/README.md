# Zigbee OTA hardware test

A manual test of `ota: - platform: zigbee` on a real board with Zigbee2MQTT (Z2M).
It is not run by CI.

| File | What it is |
|---|---|
| `zigbee-ota-test.yaml` | Test device: XIAO ESP32-C6 as a router, with uptime and chip temperature |
| `esphome_ota_test.mjs` | Z2M external converter for the test device, with `ota: true` |

The config pulls the `zigbee` component from `github://mmornati/esphome@zigbee-ota` through
`external_components`, so a recent ESPHome dev is enough. Remove that block when you build from
this branch directly.

## Steps

1. **Install the converter.** Copy `esphome_ota_test.mjs` into `<z2m data>/external_converters/`
   and restart Z2M, or register it without a restart by publishing
   `{"name": "esphome_ota_test.mjs", "code": "<file contents>"}` to
   `zigbee2mqtt/bridge/request/converter/save`.
2. **First flash, by USB.** This is the only USB flash. It also installs the bootloader with
   rollback enabled.
   ```bash
   esphome run zigbee-ota-test.yaml
   ```
   Permit join in Z2M. The device shows up as `ESPHomeOTATest`, with uptime and chip temperature.
3. **Build a second version.** Change something you can see, for example the uptime
   `update_interval` from 60s to 30s, then compile:
   ```bash
   esphome compile zigbee-ota-test.yaml
   ```
   The log ends with `Zigbee OTA file firmware.zigbee.ota: version 0x..., ... bytes`.
4. **Send the file to the device.** Publish to `zigbee2mqtt/bridge/request/device/ota_update/update`,
   either with a path Z2M can read:
   ```json
   {"id": "<friendly name>", "url": "<path as Z2M sees it>/firmware.zigbee.ota"}
   ```
   or with the file inline as hex:
   ```json
   {"id": "<friendly name>", "hex": {"data": "<hex of the file>", "file_name": "firmware.zigbee.ota"}}
   ```
   The file is `.esphome/build/zigbee-ota-test/build/firmware.zigbee.ota`. Z2M answers on
   `zigbee2mqtt/bridge/response/device/ota_update/update` when the update ends.
5. **Watch the serial log** (`esphome logs zigbee-ota-test.yaml`). The `zigbee.ota` tag logs
   `Image offered`, `Starting update`, `Progress: 10%` ... `Download complete`,
   `Image verified and selected for the next boot` and `Rebooting into the new firmware`.
   A `Block at ... skipped` line followed by `FileOffset ... set to ...` is the client recovering
   from a late duplicate block. After the reboot, Z2M shows the new file version and the device
   behaves as the second version.

With the Z2M default of 50-byte blocks, a 360 KB file takes about 30 minutes.

## Optional rollback check

Build a third version that crashes at boot, for example an `on_boot` lambda that dereferences
`nullptr`, and send it over Zigbee. The bootloader should fall back to the second version.
